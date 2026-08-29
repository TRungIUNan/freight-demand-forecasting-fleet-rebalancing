# Freight Mobility Demand Forecasting & Fleet Rebalancing Optimization
## Detailed Project Implementation Plan

**Dataset:** `Transportation & Logistics Tracking Dataset.xlsx`  
**Project type:** End-to-end Data Science + Geospatial Analytics + Time-Series Forecasting + Operations Research  
**Primary business objective:** Forecast next-day shipment demand by logistics zone and recommend vehicle repositioning between zones to reduce unmet demand and unnecessary empty travel.

---

# 1. Project framing

## 1.1 Recommended project name

**Freight Mobility Demand Forecasting & Fleet Rebalancing Optimization**

Alternative CV-friendly name:

**Logistics Demand Forecasting & Fleet Repositioning Optimization**

The term **Freight/Logistics Mobility** is more accurate than **Urban Mobility** because this dataset contains many long-haul freight trips rather than city taxi/bike/public-transport trips.

## 1.2 Core business question

At the end of day `t`, the system should answer:

1. How many shipment departures are expected from each logistics zone on day `t+1`?
2. How many observable/available vehicles are expected to be positioned in each zone?
3. Which zones will have a vehicle surplus or shortage?
4. How many vehicles should be repositioned from each surplus zone to each shortage zone?
5. What repositioning plan minimizes empty travel while preserving a high demand-fulfillment rate?

## 1.3 Final decision flow

```text
Historical Shipment Records
        |
        v
Data Cleaning + Trip Reconstruction
        |
        v
Geospatial Demand Zones
        |
        v
Daily Zone-Level Demand
        |
        v
Next-Day Demand Forecast
        |
        +--------------------+
        |                    |
        v                    v
Predicted Demand       Fleet State Reconstruction
        |                    |
        +---------+----------+
                  v
          Supply-Demand Gap
                  |
                  v
          OR-Tools Optimizer
                  |
                  v
      Vehicle Rebalancing Plan
                  |
                  v
       Backtesting / Simulation
                  |
                  v
      Decision-Support Dashboard
```

---

# 2. Important dataset facts that determine the project design

The raw workbook contains:

- **3,585 rows**
- **28 columns**
- **3,582 unique Booking IDs**
- **1,262 vehicle registrations**
- **152 raw origin-location strings**
- **116 unique origin coordinate pairs**
- **353 destination-location strings**
- **360 unique destination coordinate pairs**
- **38 vehicle types**
- **36 customers**
- **193 suppliers**
- **712 material descriptions**

Important characteristics:

- Two Booking IDs occur more than once. These repeated rows represent the same vehicle trip but contain different shipment/material/customer information. Therefore, **row count must not automatically be treated as trip count**.
- The demand target should be based on **unique trip/booking events**, not raw worksheet rows.
- The strongest continuous trip-start period is **June–August 2020**.
- Unique bookings in June–August 2020: approximately **2,867**, around 80% of the usable dataset.
- Between 2020-06-01 and 2020-08-28, trip records are present on **85 of 89 calendar days**.
- Transportation distance has a **median of 400 km**, while about **48.9%** of non-missing trips exceed 500 km and **37.2%** exceed 1,000 km.
- Therefore the project is best positioned as **freight/logistics demand forecasting**, not city-level urban mobility.

---

# 3. Scope decisions before coding

## 3.1 MVP scope

Use the following scope first:

| Component | MVP decision |
|---|---|
| Forecast horizon | Next day |
| Time granularity | Daily |
| Spatial granularity | 8–12 logistics zones, selected empirically |
| Demand target | Number of unique shipment departures per zone per day |
| Main forecasting period | June–August 2020 |
| Forecasting approach | Global model across all zones |
| Baselines | Previous day, seasonal-naive, rolling mean |
| Statistical model | Poisson Regression |
| Main ML model | LightGBM or XGBoost |
| Validation | Walk-forward / expanding-window |
| Fleet scope | Observable recurrent vehicles, not all 1,262 vehicles blindly |
| Rebalancing | Single vehicle pool for MVP |
| Solver | Google OR-Tools |
| Distance | Haversine distance between zone centroids |
| Optimization objective | Empty repositioning distance + unmet-demand penalty |
| Dashboard | Plotly/Streamlit or Power BI |
| Final evaluation | Forecast accuracy + operational simulation |

## 3.2 Explicit assumptions

Document these assumptions in the README:

1. One unique booking is treated as one shipment-departure demand unit.
2. Multiple rows sharing the same Booking ID are consolidated into one trip where trip-identifying fields agree.
3. Daily demand is modeled only on dates where data coverage is considered credible.
4. Missing days outside the core continuous period must **not** automatically be interpreted as zero demand.
5. The MVP treats one departure as requiring one vehicle assignment in the daily planning horizon.
6. Only vehicles with sufficient repeated observations should be considered part of the observable fleet for state reconstruction.
7. Current GPS snapshot fields are not automatically assumed to represent historical end-of-day vehicle positions.
8. Vehicle-type compatibility is postponed to the advanced version.
9. Haversine distance is a proxy for repositioning cost in the MVP; road-network distance is an optional upgrade.

---

# 4. Recommended repository structure

```text
freight-mobility-forecasting/
|
|-- README.md
|-- requirements.txt
|-- .gitignore
|-- configs/
|   `-- config.yaml
|
|-- data/
|   |-- raw/
|   |   `-- Transportation & Logistics Tracking Dataset.xlsx
|   |-- interim/
|   |   |-- cleaned_rows.parquet
|   |   |-- trip_level.parquet
|   |   |-- canonical_locations.parquet
|   |   `-- vehicle_history.parquet
|   |-- processed/
|   |   |-- zone_mapping.parquet
|   |   |-- daily_zone_demand.parquet
|   |   |-- forecasting_features.parquet
|   |   |-- fleet_state_daily.parquet
|   |   `-- optimization_inputs.parquet
|   `-- outputs/
|       |-- forecasts.csv
|       |-- rebalancing_plan.csv
|       `-- simulation_metrics.csv
|
|-- notebooks/
|   |-- 00_data_dictionary_review.ipynb
|   |-- 01_data_quality_audit.ipynb
|   |-- 02_cleaning_and_trip_reconstruction.ipynb
|   |-- 03_temporal_coverage_analysis.ipynb
|   |-- 04_geospatial_zone_design.ipynb
|   |-- 05_demand_dataset_construction.ipynb
|   |-- 06_demand_eda.ipynb
|   |-- 07_feature_engineering.ipynb
|   |-- 08_forecasting_baselines.ipynb
|   |-- 09_forecasting_models.ipynb
|   |-- 10_fleet_state_reconstruction.ipynb
|   |-- 11_rebalancing_optimization.ipynb
|   `-- 12_business_simulation.ipynb
|
|-- src/
|   |-- data/
|   |   |-- load.py
|   |   |-- clean.py
|   |   `-- validate.py
|   |-- features/
|   |   |-- temporal.py
|   |   |-- demand.py
|   |   `-- geospatial.py
|   |-- models/
|   |   |-- baselines.py
|   |   |-- train.py
|   |   |-- predict.py
|   |   `-- evaluate.py
|   |-- fleet/
|   |   |-- reconstruct_state.py
|   |   `-- supply_demand.py
|   |-- optimization/
|   |   |-- distance_matrix.py
|   |   `-- rebalance.py
|   `-- utils/
|       `-- helpers.py
|
|-- dashboard/
|   `-- app.py
|
|-- tests/
|   |-- test_cleaning.py
|   |-- test_features.py
|   |-- test_temporal_leakage.py
|   `-- test_optimizer.py
|
|-- docs/
|   |-- project_plan.md
|   |-- data_dictionary.md
|   `-- architecture.png
|
`-- models/
    `-- final_model.*
```

---

# 5. Phase 0 — Define the prediction contract

Before touching the model, write a short prediction contract.

## Step 0.1 — Define prediction time

Example:

> At 23:59 on day `t`, forecast shipment departures for each zone on day `t+1`.

This determines which information is legally available to the model.

## Step 0.2 — Define target

For zone `z` and day `t`:

```text
demand(z, t) = number of UNIQUE booking/trip departures
               whose valid Trip Start Date falls on day t
               and whose origin belongs to zone z
```

Do not use raw row counts before duplicate-booking consolidation.

## Step 0.3 — Define forecast unit

One ML row:

```text
(date, origin_zone)
```

Target:

```text
next_day_trip_count
```

## Step 0.4 — Define success criteria

Forecast layer:

- Must beat at least one simple baseline.
- MAE and WAPE reported overall and by zone.
- Error inspected for high-demand zones, not only global average.

Decision layer:

- Demand fulfillment improves versus no-rebalancing.
- Optimization is at least as good as a simple nearest-surplus heuristic on the chosen objective.
- Optimizer never sends more vehicles than available.
- All decision variables are non-negative integers.

**Output:** written prediction contract in README/config.

---

# 6. Phase 1 — Reproducible setup

## Step 1.1 — Create environment

Recommended libraries:

```text
pandas
numpy
scikit-learn
lightgbm or xgboost
statsmodels
geopandas
shapely
folium
plotly
ortools
pyarrow
joblib
streamlit
pytest
```

## Step 1.2 — Keep raw data immutable

Never overwrite the uploaded Excel file.

## Step 1.3 — Create a configuration file

Store:

```yaml
forecast_horizon_days: 1
core_period_start: "2020-06-01"
core_period_end: "2020-08-28"
candidate_zone_counts: [6, 8, 10, 12]
lag_days: [1, 2, 3, 7, 14]
rolling_windows: [3, 7, 14]
recurrent_vehicle_min_trips: 3
```

**Output:** reproducible project skeleton and frozen raw file.

---

# 7. Phase 2 — Raw data audit

Notebook: `01_data_quality_audit.ipynb`

## Step 2.1 — Load both sheets

Read:

- `Data Dictionary`
- `Primary Data`

## Step 2.2 — Validate schema

Check:

- expected 28 columns
- column spelling
- duplicate column names
- raw types
- row count
- unique Booking IDs

Special schema issue:

```text
Curren Location Longitude
```

contains a typo in the original column name.

Rename into a canonical snake_case schema, for example:

```text
current_location_longitude
```

## Step 2.3 — Standardize encoded missing values

Treat at least the following as missing:

```text
NULL
Null
NA
blank / None
```

Do this before calculating missingness.

## Step 2.4 — Produce quality report

For every column calculate:

- non-missing count
- missing count
- missing rate
- number of unique values
- top categories for categorical columns
- min / median / max for numeric columns
- min/max for date columns
- suspicious values

## Step 2.5 — Detect sensitive/unnecessary identifiers

Flag:

- Driver Name
- Driver Mobile No

They should not be used as predictive features.

**Output:** `data_quality_report.csv` or markdown summary.

**Done when:** every column has a documented quality status and planned treatment.

---

# 8. Phase 3 — Canonical cleaning

Notebook: `02_cleaning_and_trip_reconstruction.ipynb`

## Step 3.1 — Rename columns

Use stable snake_case names.

Example:

```text
Booking Date -> booking_date
Trip Start Date -> trip_start_datetime
Transportation Distance (KM) -> transportation_distance_km
```

## Step 3.2 — Normalize strings

Apply:

- trim leading/trailing whitespace
- normalize repeated spaces
- canonicalize missing tokens
- preserve original raw text in the raw layer

Do not blindly title-case every field; company names and registration IDs can be damaged.

## Step 3.3 — Parse datetime fields

Parse:

- booking_date
- data_ping_time
- planned_eta
- actual_eta
- trip_start_datetime
- trip_end_datetime

Create flags:

```text
invalid_booking_date
invalid_trip_start
invalid_planned_eta
invalid_actual_eta
invalid_trip_end
```

## Step 3.4 — Detect 1899 timestamp corruption

The dataset contains timestamp strings around:

```text
1899-12-30
```

These are invalid operational dates and likely originate from time-only Excel values.

Convert them to missing and retain a quality flag.

## Step 3.5 — Validate coordinates

For latitude:

```text
-90 <= latitude <= 90
```

For longitude:

```text
-180 <= longitude <= 180
```

Also inspect whether coordinates are geographically plausible for the dataset.

## Step 3.6 — Validate distance

Check:

- missing distance
- zero distance
- negative distance
- extreme distance

Observed raw-data facts:

- 148 missing values
- 18 zero-distance values among non-missing records
- median about 400 km
- maximum 2,898 km

Do not automatically drop every long-distance record because long-haul trips are characteristic of the data.

## Step 3.7 — Create data-quality flags instead of silently deleting

Example:

```text
is_valid_trip_start
is_valid_temporal_sequence
is_valid_origin_coordinate
is_valid_destination_coordinate
is_valid_distance
```

**Output:** `cleaned_rows.parquet`

---

# 9. Phase 4 — Reconstruct one row per trip

This phase is essential.

## Step 4.1 — Investigate duplicate Booking IDs

Observed:

```text
MVCV0000759/082021 -> 2 rows
MVCV0000798/082021 -> 3 rows
```

The repeated rows use the same vehicle, origin, destination and trip timing, while some material/customer/source-system fields differ.

Interpretation:

> They are likely multiple shipment/line-item records attached to the same physical trip.

## Step 4.2 — Define a trip identity

Primary key:

```text
booking_id
```

Cross-check duplicates using:

```text
vehicle_registration
trip_start_datetime
origin coordinates
destination coordinates
```

## Step 4.3 — Consolidate duplicate-booking rows

For trip-identifying fields:

- require consistency where possible
- use one canonical value

For multi-valued fields such as material/customer:

- optionally store a list
- or retain count fields such as:
  - `n_materials`
  - `n_customers`

Do not count repeated line items as extra vehicle demand.

## Step 4.4 — Produce trip-level table

Expected size should be near:

```text
3,582 unique trips
```

**Output:** `trip_level.parquet`

**Done when:** exactly one row represents one physical booking/trip.

---

# 10. Phase 5 — Temporal consistency and study-window selection

Notebook: `03_temporal_coverage_analysis.ipynb`

## Step 5.1 — Check chronology

Evaluate:

```text
booking_date <= trip_start
trip_start <= planned_eta
trip_start <= actual_eta
trip_start <= trip_end
```

Observed issues in the raw data include:

- Booking date later than trip start for a substantial subset.
- Planned ETA earlier than trip start for some rows.
- Actual ETA earlier than trip start for a small number of rows.
- Trip End Date earlier than Trip Start Date for some rows.

Do not use suspicious lead-time or duration features until their semantics are verified.

## Step 5.2 — Verify `Ontime`

For valid Planned ETA / Actual ETA pairs, check:

```text
derived_ontime = Actual ETA <= Planned ETA
```

In this dataset, the provided `Ontime` field is consistent with this rule for valid comparable records. This makes it useful for descriptive delivery-performance analysis, but it remains a post-trip variable and must not be used to predict next-day demand.

## Step 5.3 — Inspect monthly trip-start coverage

The major concentration is:

```text
2020-06: ~496 rows
2020-07: ~942 rows
2020-08: ~1,432 raw rows
```

After consolidating repeated booking IDs, June–August contains approximately **2,867 unique bookings**.

## Step 5.4 — Choose the core forecasting period

Recommended initial period:

```text
2020-06-01 through 2020-08-28
```

Why:

- dense coverage
- 85 of 89 calendar days contain trip-start records
- avoids treating long gaps elsewhere as genuine zero demand

## Step 5.5 — Create a data-coverage calendar

For each date create:

```text
date
has_any_trip_record
n_unique_bookings
coverage_flag
```

Only create explicit zero-demand zone rows on dates considered covered.

**Output:** `coverage_calendar.parquet`

---

# 11. Phase 6 — Location canonicalization

Notebook: `04_geospatial_zone_design.ipynb`

## Step 6.1 — Do not trust raw location strings as unique entities

Observed:

- 152 raw origin strings
- only 124 origin strings after simple case/space normalization
- 116 unique origin coordinate pairs

This shows text naming inconsistencies.

## Step 6.2 — Use coordinates as primary location identity

Create canonical origin point:

```text
(origin_latitude, origin_longitude)
```

Create canonical destination point similarly.

## Step 6.3 — Map raw names to canonical points

Create:

```text
raw_origin_location
canonical_origin_point_id
```

Preserve names for human-readable dashboard labels.

## Step 6.4 — Inspect coordinate duplicates and naming mismatches

Check:

- same coordinates with different names
- same name with different coordinates
- coordinates too close to justify different zones

**Output:** `canonical_locations.parquet`

---

# 12. Phase 7 — Design logistics demand zones

The dataset is too sparse to forecast 152 raw origin names independently.

## Step 7.1 — Candidate methods

Test:

1. K-Means on projected coordinates
2. Agglomerative clustering with geographic distance
3. DBSCAN/HDBSCAN with Haversine distance

Recommended MVP:

> Compare a simple geographically sensible clustering approach across several zone counts before introducing a more complex method.

## Step 7.2 — Candidate zone counts

Evaluate:

```text
K = 6, 8, 10, 12
```

## Step 7.3 — Evaluate each zone solution

Do not select only by silhouette score.

Evaluate:

### Geographic quality
- compactness
- visually sensible grouping
- centroid locations

### Forecasting viability
- trips per zone
- percentage of zone-days with zero demand
- minimum history per zone
- extreme dominant zones

### Business usability
- number of zones understandable on a map
- no tiny zones with only a handful of departures

## Step 7.4 — Select final zone count

Recommended expected range:

```text
8–12 zones
```

but allow evidence to choose otherwise.

## Step 7.5 — Save mapping

Each canonical location gets:

```text
location_id
zone_id
zone_centroid_lat
zone_centroid_lon
```

**Output:** `zone_mapping.parquet`

**Done when:** zones are spatially coherent and contain enough demand history for forecasting.

---

# 13. Phase 8 — Construct the daily zone-demand panel

Notebook: `05_demand_dataset_construction.ipynb`

## Step 8.1 — Assign origin zone to each trip

Join trip-level data to `zone_mapping`.

## Step 8.2 — Create daily target

For each date and zone:

```text
demand = number of unique booking IDs starting in that zone on that date
```

## Step 8.3 — Create complete zone-date grid

For covered dates only:

```text
all selected dates x all active zones
```

Fill absent zone-day combinations with zero only when the day itself has credible dataset coverage.

## Step 8.4 — Add basic static zone descriptors

Possible fields:

```text
zone_id
zone_centroid_lat
zone_centroid_lon
historical_mean_demand
historical_median_demand
```

Avoid computing full-history statistics before splitting train/test; static statistics used in modeling must be generated point-in-time.

**Output:** `daily_zone_demand.parquet`

---

# 14. Phase 9 — Demand EDA

Notebook: `06_demand_eda.ipynb`

Answer:

1. Which zones contribute most demand?
2. Is demand concentrated geographically?
3. What is the day-of-week pattern?
4. How volatile is each zone?
5. Which zones have many zeros?
6. Is there visible weekly seasonality?
7. Are there demand spikes?
8. How does Regular vs Market shipment mix vary?
9. Does average trip distance differ by origin zone?
10. Are there strong customer-origin relationships?

Recommended visualizations:

- daily total trips line chart
- zone-demand heatmap: date × zone
- demand distribution by zone
- weekday boxplots
- map of trip origins
- origin-zone centroid map
- OD flow summary
- rolling 7-day demand
- Regular vs Market share

Do not make modeling decisions from charts only; record quantitative evidence.

---

# 15. Phase 10 — Point-in-time feature engineering

Notebook: `07_feature_engineering.ipynb`

## Step 10.1 — Calendar features

```text
day_of_week
day_of_month
week_of_year
month
is_weekend
```

Because the core period is short, avoid overloading the model with calendar variables that barely vary.

## Step 10.2 — Lag features

Recommended:

```text
demand_lag_1
demand_lag_2
demand_lag_3
demand_lag_7
demand_lag_14
```

## Step 10.3 — Rolling features

Recommended:

```text
rolling_mean_3
rolling_mean_7
rolling_mean_14
rolling_std_7
rolling_max_7
```

CRITICAL:

```python
historical_series.shift(1).rolling(window)
```

The current target must never appear inside its own rolling feature.

## Step 10.4 — Expanding historical zone features

Possible:

```text
expanding_mean_demand
expanding_std_demand
days_since_last_nonzero_demand
```

All computed from data strictly earlier than the prediction date.

## Step 10.5 — Optional historical composition features

Only after the basic model is stable:

```text
historical_regular_share
historical_avg_trip_distance
historical_customer_concentration
```

Use lagged/rolling history, not information belonging to tomorrow's shipment rows.

## Step 10.6 — Zone encoding

For tree models:

```text
zone_id
```

can be categorical/encoded.

For Poisson Regression:

- one-hot encode zones
- scale numeric features where appropriate

**Output:** `forecasting_features.parquet`

---

# 16. Phase 11 — Leakage-control checklist

At prediction time for day `t+1`, the following direct values must not be used:

```text
Actual ETA of t+1 trips
Ontime of t+1 trips
Current Location snapshot from t+1 trips
Trip End Date of t+1 trips
Destination of trips that have not yet been observed
Material Shipped of future trips
Customer/Supplier identity of future departures
Vehicle Registration assigned to future departures
```

Possible use:

- historical aggregates from earlier dates
- known calendar information
- lagged demand
- lagged fleet state
- operational information genuinely available before prediction time

Create automated tests to ensure feature dates are always earlier than target dates.

---

# 17. Phase 12 — Forecasting baselines

Notebook: `08_forecasting_baselines.ipynb`

A model has value only if it beats meaningful simple rules.

## Baseline A — Previous-day naive

```text
forecast(t) = demand(t-1)
```

## Baseline B — Seasonal naive

```text
forecast(t) = demand(t-7)
```

## Baseline C — 7-day rolling mean

```text
forecast(t) = mean(demand(t-1 ... t-7))
```

Evaluate all baselines with the same time splits as the final model.

**Output:** baseline leaderboard.

---

# 18. Phase 13 — Statistical model

Notebook: `09_forecasting_models.ipynb`

## Step 13.1 — Poisson Regression

Reason:

Demand is non-negative count data.

Use:

- zone categorical variables
- calendar features
- lag features

Check:

- non-negative predictions
- overdispersion
- residual patterns

If severe overdispersion is present, consider Negative Binomial only as an optional comparison.

---

# 19. Phase 14 — Main ML model

Recommended candidates:

1. LightGBM Regressor
2. LightGBM with Poisson objective
3. XGBoost Regressor

Do **not** make LSTM/GRU/Transformer the main model because the dataset is small and the continuous history is short.

## Step 14.1 — Global model

Train one model across all zones rather than one model per zone.

Reason:

```text
small number of dates per zone
+
shared demand patterns
=
global model is more data-efficient
```

## Step 14.2 — Minimal hyperparameter tuning

Tune only high-impact parameters, for example:

```text
learning_rate
n_estimators
max_depth / num_leaves
min_child_samples
subsample
colsample_bytree
```

Do not optimize against the final test period.

---

# 20. Phase 15 — Time-series validation

Never use random shuffled train/test split.

Use expanding-window or walk-forward validation.

Example:

```text
Fold 1:
Train  -> early June
Valid  -> later June

Fold 2:
Train  -> June + early July
Valid  -> later July

Fold 3:
Train  -> June + July + early August
Valid  -> later August
```

Exact dates should be chosen after the final core-period and minimum-lag requirements are fixed.

Keep a final untouched test block.

---

# 21. Phase 16 — Forecast metrics and diagnostics

Primary metrics:

## MAE

```text
mean absolute number of trips missed per zone-day
```

## WAPE

```text
sum(abs(actual - forecast)) / sum(actual)
```

Useful because demand volume differs by zone.

## RMSE

Use as a secondary metric to expose large misses.

Also report:

- MAE by zone
- WAPE by zone where denominator is meaningful
- high-demand-day error
- prediction bias:

```text
mean(prediction - actual)
```

A model that systematically underpredicts demand can damage fleet planning even with reasonable average error.

---

# 22. Phase 17 — Model selection

Select the final model based on:

1. walk-forward performance
2. stability across folds
3. improvement over seasonal-naive
4. error on high-demand zones
5. operational downstream performance

Do not select the final model only because it has the smallest overall RMSE.

Save:

```text
final_model
feature_list
training_period
validation_results
model_version
```

---

# 23. Phase 18 — Explainability

Use explainability to answer:

- Is yesterday's demand important?
- Is seven-day lag important?
- Which zones behave differently?
- Is day-of-week informative?

For tree models:

- feature importance
- SHAP summary
- SHAP for selected high-demand predictions

Avoid presenting SHAP as causal evidence.

---

# 24. Phase 19 — Fleet state reconstruction

Notebook: `10_fleet_state_reconstruction.ipynb`

This phase turns trip history into an estimate of vehicle supply.

## Step 19.1 — Define observable fleet

The dataset has 1,262 unique vehicle registrations, but many appear only once.

Approximate recurrent counts after booking consolidation:

```text
>= 2 trips: 578 vehicles
>= 3 trips: 372 vehicles
>= 5 trips: 231 vehicles
>= 10 trips: 65 vehicles
```

Recommended MVP:

```text
recurrent_vehicle_min_trips = 3
```

Call this:

> **observable recurrent fleet**

Do not claim that all vehicles are company-owned.

## Step 19.2 — Sort each vehicle's trips chronologically

For each vehicle:

```text
trip 1 -> trip 2 -> trip 3 -> ...
```

## Step 19.3 — Determine completed-trip destination

Preferred completion timestamp:

1. valid `Actual ETA`
2. fallback to another verified completion/arrival field only if its semantics are reliable

Do not automatically trust corrupted timestamps.

## Step 19.4 — Estimate end-of-day location

If a vehicle completed its last known trip before the planning cutoff:

```text
vehicle_location = destination_zone
```

If still in transit:

```text
available = False
```

## Step 19.5 — Validate sequential movements

For repeated vehicles compare:

```text
previous destination
next origin
```

Measure how often they are geographically consistent.

Large mismatches indicate that unobserved trips exist between records, which limits exact fleet-state inference.

## Step 19.6 — Produce daily fleet state

```text
date
vehicle_registration
estimated_zone
available_flag
confidence_flag
```

Aggregate:

```text
date
zone
available_vehicles
```

**Output:** `fleet_state_daily.parquet`

---

# 25. Phase 20 — Fleet-state confidence

Because the dataset is not a complete telematics history, create confidence tiers.

Example:

### High confidence
- recurrent vehicle
- valid arrival timestamp
- consistent previous destination / next origin

### Medium confidence
- recurrent vehicle
- valid arrival
- occasional sequence gap

### Low confidence
- single observation
- corrupted time
- unexplained large location jump

For the MVP optimizer, consider using only High + Medium confidence vehicles.

This makes the analysis more defensible.

---

# 26. Phase 21 — Supply-demand gap

For planning date `t+1`:

```text
forecast_demand(zone)
available_supply(zone)
```

Calculate:

```text
gap = available_supply - forecast_demand
```

Interpretation:

```text
gap > 0  -> surplus
gap < 0  -> shortage
gap = 0  -> balanced
```

Create:

```text
zone
forecast_demand
available_supply
surplus
shortage
```

---

# 27. Phase 22 — Zone distance matrix

Notebook: `11_rebalancing_optimization.ipynb`

## Step 22.1 — Compute zone centroids

Each zone has:

```text
centroid_latitude
centroid_longitude
```

## Step 22.2 — Calculate Haversine distance

For every origin-zone / destination-zone pair:

```text
distance_km[i, j]
```

Set:

```text
distance(i, i) = 0
```

## Step 22.3 — Sanity-check matrix

Validate:

```text
distance(i,j) >= 0
distance(i,j) ~= distance(j,i)
```

**Output:** `zone_distance_matrix.csv`

---

# 28. Phase 23 — Rebalancing optimization formulation

Decision variable:

```text
x[i,j] = integer number of vehicles repositioned
         from surplus zone i to shortage zone j
```

Shortage variable:

```text
u[j] = unmet vehicle demand remaining in zone j
```

## Objective

Minimize:

```text
sum(distance[i,j] * x[i,j])
+
lambda * sum(u[j])
```

Interpretation:

- first term penalizes empty driving
- second term strongly penalizes unserved demand

## Supply constraint

For each surplus zone:

```text
sum_j x[i,j] <= surplus[i]
```

## Demand constraint

For each shortage zone:

```text
sum_i x[i,j] + u[j] >= shortage[j]
```

## Integrality

```text
x[i,j] >= 0 and integer
u[j] >= 0 and integer
```

## Optional maximum repositioning radius

Advanced constraint:

```text
x[i,j] = 0 if distance[i,j] > max_reposition_km
```

This prevents unrealistic cross-country empty repositioning.

---

# 29. Phase 24 — Build comparison strategies

Notebook: `12_business_simulation.ipynb`

Backtest the operational decision layer using actual demand.

Compare at least:

## Strategy A — No rebalancing

Vehicles stay where they are.

## Strategy B — Greedy nearest-zone heuristic

Repeatedly move surplus vehicles to the nearest shortage zone.

## Strategy C — Global optimization

Use OR-Tools.

This comparison is necessary to prove that optimization adds value.

---

# 30. Phase 25 — Operational metrics

For each test day calculate:

## Demand fulfillment rate

```text
served_demand / total_actual_demand
```

## Unmet demand

```text
actual demand not served
```

## Empty repositioning distance

```text
sum(repositioned vehicles * zone distance)
```

## Vehicles repositioned

```text
sum(x[i,j])
```

## Vehicle utilization proxy

```text
vehicles assigned to demand / available vehicles
```

## Optional composite objective

```text
empty_km_cost
+ unmet_demand_penalty
```

Report average, median and worst-day performance.

---

# 31. Phase 26 — Forecast-vs-perfect-information experiment

This is a strong Data Science experiment.

Run the optimizer twice:

### Scenario 1 — Forecast-driven

Use predicted demand.

### Scenario 2 — Oracle / perfect-information upper bound

Use actual next-day demand.

Difference:

```text
oracle operational performance
-
forecast-driven operational performance
```

This quantifies how much value is lost because of forecast error.

It links ML accuracy directly to business decisions.

---

# 32. Phase 27 — Sensitivity analysis

Test how rebalancing changes when:

- unmet-demand penalty changes
- maximum repositioning distance changes
- recurrent-vehicle threshold changes
- zone count changes
- forecast model changes

Recommended experiments:

```text
lambda: low / medium / high
max reposition distance: 100 / 250 / 500 km
vehicle min trips: 2 / 3 / 5
zone count: 6 / 8 / 10 / 12
```

Document which conclusions are robust.

---

# 33. Phase 28 — Advanced version: vehicle types

Only after the single-fleet MVP is stable.

The raw dataset has 38 vehicle-type labels and about 21.3% missing values.

Create broader classes, for example:

```text
Light
Medium
Heavy
Trailer
Multi-Axle
Unknown
```

Then forecast/optimize by:

```text
zone x vehicle_class
```

Add compatibility constraints:

```text
vehicle class must satisfy required shipment class
```

Do not do this first; sparse data may become a major issue.

---

# 34. Phase 29 — Advanced version: external/market fleet

Shipment Type contains:

```text
Regular: 3,527 raw rows
Market: 58 raw rows
```

The Market class is very small.

A future extension can model:

```text
owned/recurrent fleet
+
external market capacity
```

and add:

```text
external_hiring_cost
```

to the optimization objective.

Treat this as an extension, not an MVP requirement.

---

# 35. Phase 30 — Dashboard design

Recommended pages:

## Page 1 — Network overview

KPIs:

- Unique trips
- Active/recurrent vehicles
- Average/median trip distance
- Active logistics zones
- Regular vs Market share

Visuals:

- origin map
- top OD routes
- daily trip volume

## Page 2 — Demand forecasting

KPIs:

- next-day predicted demand
- MAE
- WAPE
- highest-demand zone

Visuals:

- actual vs forecast
- zone × date heatmap
- forecast by zone
- model error by zone

## Page 3 — Fleet supply and gap

For every zone show:

```text
predicted demand
available vehicles
surplus/shortage
```

Use a map for geographic interpretation.

## Page 4 — Rebalancing recommendations

Show:

```text
source zone
destination zone
vehicles to move
distance
```

KPIs:

- expected fulfillment
- unmet demand
- empty km
- vehicles repositioned

## Page 5 — Scenario comparison

Compare:

```text
No Rebalancing
Greedy
Optimization
Oracle
```

---

# 36. Phase 31 — Engineering quality

## Unit tests

At minimum:

### Cleaning tests
- encoded missing tokens converted properly
- invalid 1899 timestamps rejected
- coordinate rules correct

### Trip tests
- one row per Booking ID after consolidation
- duplicate booking consolidation does not create extra demand

### Feature tests
- lag feature for date `t` uses only `< t`
- rolling window excludes current target

### Optimization tests
- no negative allocations
- integer allocations
- source outflow never exceeds supply
- reported shortages match constraints

---

# 37. Phase 32 — Reproducible pipeline

After notebooks are stable, move logic into `src/`.

Target CLI-style flow:

```text
01 load
02 clean
03 build trip table
04 create zones
05 build daily demand
06 generate features
07 train model
08 forecast
09 reconstruct fleet
10 optimize
11 simulate
12 export dashboard data
```

Notebooks should become explanation/analysis layers rather than the only place where logic exists.

---

# 38. Phase 33 — Final README

Recommended README structure:

1. Problem statement
2. Dataset
3. Why freight mobility, not urban mobility
4. Data-quality findings
5. Architecture
6. Geographic zone creation
7. Forecasting methodology
8. Time-series validation
9. Fleet-state reconstruction
10. Optimization formulation
11. Results
12. Business impact
13. Limitations
14. Future work
15. How to run the project

Include only measured results. Never invent percentage improvements before the simulation is complete.

---

# 39. Phase 34 — CV-ready output

After the project is finished, convert measured results into three bullets.

Template:

```text
- Built a geospatial demand-forecasting pipeline that transformed shipment-level
  records into daily logistics-zone demand and predicted next-day freight activity
  using lag, rolling and calendar features.

- Developed and validated global demand-forecasting models with walk-forward
  evaluation, improving [METRIC] by [X%] versus [BASELINE].

- Formulated an OR-Tools fleet-rebalancing optimizer that increased simulated
  demand fulfillment from [A%] to [B%] while reducing empty repositioning
  distance by [C%] versus [COMPARISON STRATEGY].
```

Fill `[X]`, `[A]`, `[B]`, `[C]` only with final measured values.

---

# 40. Recommended order of execution

Do not jump directly to the ML model.

Follow this order:

```text
1. Data dictionary review
2. Data-quality audit
3. Datetime cleaning
4. Duplicate-booking investigation
5. Trip-level reconstruction
6. Temporal coverage analysis
7. Location canonicalization
8. Geospatial zone design
9. Daily zone-demand panel
10. Demand EDA
11. Point-in-time feature engineering
12. Baselines
13. Walk-forward validation
14. Poisson model
15. LightGBM/XGBoost
16. Model diagnostics
17. Observable-fleet definition
18. Fleet-state reconstruction
19. Supply-demand gap
20. Zone distance matrix
21. OR-Tools optimization
22. No-rebalance / heuristic / optimizer backtest
23. Sensitivity analysis
24. Dashboard
25. Refactor notebooks into src modules
26. Tests
27. README
28. CV bullets
```

---

# 41. Project milestones and Definition of Done

## Milestone 1 — Clean trip table

Done when:

- all 28 original columns are documented
- missing tokens standardized
- invalid timestamps flagged
- duplicate Booking IDs understood and consolidated
- one row corresponds to one physical booking/trip

## Milestone 2 — Forecast-ready demand table

Done when:

- final demand zones are selected
- complete covered date × zone panel exists
- target is unique-booking demand
- zero-demand values are created only on credible coverage dates

## Milestone 3 — Valid forecasting model

Done when:

- random split is not used
- baseline leaderboard exists
- walk-forward validation exists
- final model beats or clearly contextualizes simple baselines
- no target leakage is found

## Milestone 4 — Valid fleet state

Done when:

- recurrent-fleet rule is documented
- vehicle sequences are checked
- availability logic is reproducible
- fleet-confidence limitations are quantified

## Milestone 5 — Optimization system

Done when:

- supply and demand are connected
- OR-Tools returns feasible integer allocations
- empty-distance and shortage objectives are reported
- no-rebalance and heuristic baselines are implemented

## Milestone 6 — Portfolio release

Done when:

- business simulation has measured outcomes
- dashboard is reproducible
- README explains assumptions and limitations
- repository contains clean code, not only notebooks
- CV bullets use real measured impact

---

# 42. Main risks and mitigation

| Risk | Why it matters | Mitigation |
|---|---|---|
| Short forecasting history | Weak long-term seasonality learning | Focus on next-day forecasting and global models |
| Non-continuous historical coverage | Missing dates may look like zero demand | Restrict modeling to verified dense period |
| Repeated Booking IDs | Raw row counts can overstate demand | Reconstruct one trip per unique booking |
| Location-name inconsistency | Artificially splits the same location | Canonicalize using coordinates |
| Temporal corruption | Can create invalid durations/leakage | Quality flags and strict point-in-time rules |
| Many one-off vehicles | Cannot infer complete fleet state | Use observable recurrent fleet |
| Current-location snapshots | May not represent end-of-day historical state | Prefer completed-trip destination logic |
| Vehicle Type missing 21% | Multi-type optimization becomes sparse | Start with single-fleet MVP |
| Market shipments very rare | Poor basis for separate Market model | Keep as descriptive/advanced feature |
| Haversine != road distance | Reposition cost is approximate | State limitation; optionally upgrade later |

---

# 43. What makes this project strong for a Data Scientist portfolio

This project demonstrates a sequence of decisions rather than one isolated model:

```text
messy operational data
        ->
spatial abstraction
        ->
time-series forecasting
        ->
point-in-time ML validation
        ->
fleet-state inference
        ->
mathematical optimization
        ->
business simulation
        ->
decision-support visualization
```

The strongest story is not "I trained XGBoost."

The strongest story is:

> **I converted imperfect logistics tracking data into a forecast-driven operational decision system and measured how model predictions affected fleet allocation decisions.**
