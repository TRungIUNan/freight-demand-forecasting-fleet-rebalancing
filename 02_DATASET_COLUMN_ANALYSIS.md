# Transportation & Logistics Tracking Dataset
## Detailed Column Analysis for Freight Mobility Demand Forecasting & Fleet Rebalancing

**Workbook:** `Transportation & Logistics Tracking Dataset.xlsx`  
**Sheets inspected:**

- `Data Dictionary`
- `Primary Data`

**Primary Data dimensions:** 3,585 rows × 28 columns

---

# 1. Executive data-quality summary

## Key counts

| Metric | Observed value |
|---|---:|
| Raw rows | 3,585 |
| Columns | 28 |
| Unique Booking IDs | 3,582 |
| Unique Vehicle Registrations | 1,262 |
| Raw Origin Location strings | 152 |
| Origin strings after simple case/space normalization | 124 |
| Unique origin coordinate pairs | 116 |
| Raw Destination Location strings | 353 |
| Unique destination coordinate pairs | 360 |
| Vehicle Types | 38 |
| Customers | 36 |
| Suppliers | 193 |
| Material descriptions | 712 |

## Critical findings

1. **Raw rows are not identical to physical trips.** Two Booking IDs repeat:
   - `MVCV0000759/082021` appears 2 times.
   - `MVCV0000798/082021` appears 3 times.

   Their vehicle/origin/destination/timing fields are essentially the same, while material/customer/source information can differ. The project should therefore build a **trip-level table with one row per Booking ID** before creating demand counts.

2. Missing values are encoded inconsistently as:
   - `NULL`
   - `Null`
   - `NA`
   - blank/None

3. Some datetime fields contain invalid values around **1899-12-30**, which are likely Excel time-only artifacts and must be converted to missing/invalid flags.

4. Origin naming is inconsistent. There are 152 raw origin strings but only 124 after basic text normalization and 116 coordinate pairs. Geographic coordinates should be preferred for spatial identity.

5. Several operational timestamps are temporally inconsistent. These fields must be validated before calculating durations or reconstructing vehicle state.

6. The dataset is highly concentrated in June–August 2020, making it more appropriate for **short-horizon next-day forecasting** than long-term forecasting.

7. Driver-identifying fields should not be used in predictive modeling.

---

# 2. Missing-value summary

Encoded missing tokens were counted as missing.

| Column | Missing | Missing rate |
|---|---:|---:|
| Gps Provider | 1 | 0.03% |
| Booking ID | 0 | 0.00% |
| Shipment Type | 0 | 0.00% |
| Booking Date | 0 | 0.00% |
| Vehicle Registration | 0 | 0.00% |
| Origin Location | 0 | 0.00% |
| Destination Location | 0 | 0.00% |
| Origin Location Latitude | 0 | 0.00% |
| Origin Location Longitude | 0 | 0.00% |
| Destination Location Latitude | 0 | 0.00% |
| Destination Location Longitude | 0 | 0.00% |
| Data Ping time | 1 | 0.03% |
| Planned ETA | 0 | 0.00%* |
| Current Location | 12 | 0.33% |
| Actual ETA | 25 | 0.70%* |
| Current Location Latitude | 1 | 0.03% |
| Curren Location Longitude | 1 | 0.03% |
| Ontime | 0 | 0.00% |
| Trip Start Date | 0 | 0.00%* |
| Trip End Date | 0 | 0.00%* |
| Transportation Distance (KM) | 148 | 4.13% |
| Vehicle Type | 764 | 21.31% |
| Minimum Kms To Be Covered In A Day | 2,645 | 73.78% |
| Driver Name | 317 | 8.84% |
| Driver Mobile No | 1,024 | 28.56% |
| Customer Name | 0 | 0.00% |
| Supplier Name | 0 | 0.00% |
| Material Shipped | 0 | 0.00% |

`*` Some date columns also contain invalid 1899 timestamps that are syntactically present but should be treated as unusable.

---

# 3. Temporal-data quality

## Observed valid ranges

| Field | Approximate valid range |
|---|---|
| Booking Date | 2019-04-15 to 2020-12-03 |
| Data Ping time | 2019-06-14 to 2020-08-28 |
| Planned ETA | 2019-11-30 to 2020-12-05, excluding invalid 1899 values |
| Actual ETA | 2019-12-11 to 2020-08-29, excluding invalid 1899 values |
| Trip Start Date | 2019-11-29 to 2020-12-03, excluding invalid 1899 values |
| Trip End Date | 2020-01-01 to 2020-08-28, excluding invalid 1899 values |

## Invalid 1899 values

Observed in:

- Planned ETA: 2 rows
- Actual ETA: 2 rows in addition to missing values
- Trip Start Date: 2 rows
- Trip End Date: 2 rows

Interpretation:

> `1899-12-30` is typically produced when Excel stores only a time portion without a valid calendar date.

Recommended action:

```text
convert to NaT / missing
+
create invalid_datetime_flag
```

## Temporal consistency checks

Among rows with comparable non-1899 dates:

| Rule | Violations |
|---|---:|
| Booking Date > Trip Start Date | 714 / 3,583 (~19.9%) |
| Planned ETA < Trip Start Date | 43 / 3,583 (~1.2%) |
| Actual ETA < Trip Start Date | 7 / 3,558 (~0.2%) |
| Trip End Date < Trip Start Date | 57 / 3,583 (~1.6%) |

These inconsistencies mean that lead time and trip duration should **not** be used blindly.

## Ontime consistency

For 3,558 comparable valid Planned ETA / Actual ETA pairs:

```text
Ontime = Yes  <=> Actual ETA <= Planned ETA
Ontime = No   <=> Actual ETA > Planned ETA
```

No mismatch was observed in those comparable records.

This suggests `Ontime` is derived directly from the ETA comparison.

---

# 4. Transportation-distance profile

For non-missing `Transportation Distance (KM)`:

| Statistic | Value |
|---|---:|
| Non-missing | 3,437 |
| Missing | 148 |
| Minimum | 0 km |
| Q1 | 107 km |
| Median | 400 km |
| Mean | ~841.1 km |
| Q3 | 1,290 km |
| Maximum | 2,898 km |
| Zero-distance records | 18 |
| > 500 km | ~48.9% |
| > 1,000 km | ~37.2% |

Implication:

> The dataset is dominated by substantial inter-city/long-haul movement. This supports the project name **Freight Mobility / Logistics Demand Forecasting**, not urban mobility.

---

# 5. Detailed analysis of every column

---

## 5.1 `Gps Provider`

**Data Dictionary meaning:** GPS service provider tracking the vehicle.

**Observed:**

- 28 distinct non-missing providers
- 1 missing value
- Top providers:
  - Consent Track: 2,071
  - Vamosys: 746
  - Ekta: 308
  - Krc Logistics: 172
  - Jtech: 58

**Data type:** Categorical text.

**Potential use:**

- data-quality diagnostics
- tracking-source comparison
- GPS-coverage investigation

**Do not use as a core demand-forecast feature** unless there is a clear operational reason. It may encode data-collection process rather than real customer demand.

**Cleaning:**

- trim whitespace
- normalize case only if necessary
- convert `Null` to missing

**Project role:** Secondary / diagnostic.

**Leakage risk:** Low direct temporal leakage, but high risk of **spurious source-system correlation**.

**Recommended MVP action:** Keep in cleaned data, exclude from primary forecasting feature set.

---

## 5.2 `Booking ID`

**Data Dictionary meaning:** Unique identifier for each booking.

**Observed:**

- 3,585 raw rows
- 3,582 unique Booking IDs
- Repeated IDs:
  - `MVCV0000759/082021`: 2 rows
  - `MVCV0000798/082021`: 3 rows

**Important interpretation:**

The column is described as unique, but repeated records exist. Duplicate-ID rows have matching physical-trip fields but can have different:

- GPS Provider
- Customer Name
- Material Shipped

Therefore a Booking ID appears to represent a trip/booking that may contain multiple line items.

**Primary project use:**

- canonical trip key
- demand counting
- duplicate consolidation

**Critical rule:**

```text
Demand must count unique Booking IDs, not worksheet rows.
```

**Cleaning:**

1. group by Booking ID
2. verify consistency of vehicle, origin, destination and time
3. consolidate to one trip record
4. optionally preserve list/count of materials/customers

**Project role:** Essential identifier.

**Leakage risk:** Do not feed raw Booking ID into ML.

**Recommended MVP action:** Use as key, then exclude from model features.

---

## 5.3 `Shipment Type`

**Data Dictionary meaning:** Market spot booking or Regular contract-based trip.

**Observed categories:**

- Regular: 3,527 raw rows
- Market: 58 raw rows

**Data type:** Binary categorical.

**Issue:** Severe class imbalance; Market shipments are only a small fraction.

**Potential use:**

- descriptive EDA
- historical share by zone
- advanced fleet sourcing logic
- future owned-vs-external capacity hypothesis

**Forecasting caution:**

If forecasting tomorrow's demand before tomorrow's bookings are known, tomorrow's shipment type cannot be used directly.

Safe alternative:

```text
lagged_regular_share
historical_market_share
```

**Project role:** Useful secondary business feature.

**Recommended MVP action:** EDA + historical aggregate only.

---

## 5.4 `Booking Date`

**Data Dictionary meaning:** Date and time when the booking was created.

**Observed range:** 2019-04-15 to 2020-12-03.

**Data type:** Datetime.

**Quality issue:**

For about 714 of 3,583 comparable records, Booking Date is later than Trip Start Date.

This is unexpectedly high and suggests one of the following:

- field semantics differ from assumed creation time
- system timestamps were updated later
- source-system inconsistencies
- data-entry quality issue

**Potential use:**

- booking-to-departure lead-time analysis only after investigation
- defining known demand if building a booking-aware forecast in an advanced version

**Do not use immediately for:**

```text
booking lead time
booking backlog
future known-demand feature
```

without resolving chronology.

**Project role:** Important but quality-sensitive.

**Recommended MVP action:** Keep; flag chronology; exclude lead-time features initially.

---

## 5.5 `Vehicle Registration`

**Data Dictionary meaning:** Registration number of the transport vehicle.

**Observed:**

- 1,262 unique vehicles
- most frequent vehicle appears 28 times
- after trip consolidation, approximate recurrent counts:
  - >=2 trips: 578 vehicles
  - >=3 trips: 372 vehicles
  - >=5 trips: 231 vehicles
  - >=10 trips: 65 vehicles

**Data type:** High-cardinality identifier.

**Primary use:**

- reconstruct vehicle trip sequences
- estimate fleet position and availability
- define recurrent/observable fleet

**Do not use raw registration as a demand-forecast feature.**

**Important limitation:**

A vehicle appearing in the dataset does not prove it is company-owned or continuously observable.

Use wording:

> observable recurrent fleet

not:

> the company's 1,262-truck fleet

**Project role:** Essential for fleet layer.

**Recommended MVP action:** Use for state reconstruction, exclude from forecast model.

---

## 5.6 `Origin Location`

**Data Dictionary meaning:** Initial point from which the shipment starts.

**Observed:**

- 152 raw strings
- 124 strings after simple lower-case / whitespace normalization
- top raw values include:
  - Shive, pune, maharashtra
  - Daimler India Commercial Vehicles,Kanchipuram,Tamil Nadu
  - Jamalpur, gurgaon, haryana
  - Khorajnanoda, ahmedabad, gujarat
  - Ashok Leyland Plant 2-Hosur,Hosur,Karnataka

**Issue:**

Text naming is inconsistent in capitalization and formatting.

Example conceptually:

```text
Shive, pune, maharashtra
Shive, Pune, Maharashtra
```

may refer to the same place.

**Primary use:**

- readable label
- geospatial canonicalization
- origin-zone assignment

**Do not forecast at all 152 raw text labels** because history becomes sparse.

**Preferred identity:** Origin latitude/longitude.

**Project role:** Essential business label; coordinates should drive clustering.

---

## 5.7 `Destination Location`

**Data Dictionary meaning:** Final destination where the shipment should be delivered.

**Observed:**

- 353 distinct raw strings
- common destinations include Kanchipuram/Pune/Ahmedabad locations

**Data type:** High-cardinality categorical text.

**Potential use:**

- OD-flow EDA
- trip-network visualization
- vehicle destination-zone reconstruction
- historical destination-mix features

**Forecasting leakage warning:**

If destination for tomorrow's trips is not known at prediction time, it cannot be used directly to predict tomorrow's trip count.

**Project role:** Essential for fleet-state reconstruction, secondary for demand forecasting.

**Recommended MVP action:** Map to destination zone; do not use future destination directly in demand model.

---

## 5.8 `Origin Location Latitude`

**Data Dictionary meaning:** Latitude of origin.

**Observed:**

- no encoded missing values
- 115 distinct latitude values
- range: approximately 9.974 to 30.000

**Data type:** Numeric geographic coordinate.

**Primary use:**

- canonical origin point
- geospatial clustering
- mapping
- zone centroid creation

**Cleaning:**

- numeric conversion
- range validation
- pair with origin longitude
- do not independently cluster latitude without longitude

**Project role:** Essential.

---

## 5.9 `Origin Location Longitude`

**Data Dictionary meaning:** Longitude of origin.

**Observed:**

- no encoded missing values
- 116 distinct longitude values
- range: approximately 72.056 to 91.844

Combined with latitude:

- **116 unique origin coordinate pairs**

**Primary use:** Same as origin latitude.

**Project role:** Essential.

---

## 5.10 `Destination Location Latitude`

**Data Dictionary meaning:** Latitude of destination.

**Observed:**

- no encoded missing values
- 358 distinct latitude values
- range: approximately 8.173 to 32.685

**Primary use:**

- destination-zone mapping
- OD analysis
- vehicle-state reconstruction

**Project role:** Essential for fleet layer.

---

## 5.11 `Destination Location Longitude`

**Data Dictionary meaning:** Longitude of destination.

**Observed:**

- no encoded missing values
- 355 distinct longitude values
- range: approximately 70.741 to 94.961

Combined destination coordinates form approximately:

- **360 unique coordinate pairs**

The number of coordinate pairs is slightly greater than destination-name count, indicating that the same named destination can sometimes have different recorded points.

**Project role:** Essential for fleet layer.

---

## 5.12 `Data Ping time`

**Data Dictionary meaning:** Timestamp of the last recorded GPS ping.

**Observed:**

- 1 encoded missing `NULL`
- approximately 2,588 distinct non-missing timestamps
- valid range around 2019-06-14 to 2020-08-28

**Data type:** Datetime.

**Potential use:**

- tracking freshness
- data-quality analysis
- current-location recency

**Caution:**

This is a GPS snapshot timestamp, not necessarily a trip-start or trip-end event.

It should not be assumed to represent end-of-day fleet state.

**Project role:** Diagnostic / optional fleet-confidence feature.

**Forecast leakage risk:** High if a future GPS ping is used for an earlier forecast cutoff.

**Recommended MVP action:** Do not use in forecasting; use only with strict cutoff logic if needed for fleet confidence.

---

## 5.13 `Planned ETA`

**Data Dictionary meaning:** Expected destination arrival time according to the trip plan.

**Observed:**

- two invalid 1899 time-only values
- otherwise valid dates extend roughly from 2019-11-30 to 2020-12-05
- 43 comparable rows have Planned ETA earlier than Trip Start Date

**Potential use:**

- planned trip duration
- expected vehicle availability
- service performance analysis

**Caution:**

Use only if the timestamp is valid and chronology is plausible.

**Forecasting leakage:**

A planned ETA for an already-known active trip can be legitimate for fleet supply planning, but a future trip's Planned ETA cannot be used if the trip itself is not known at forecast time.

**Project role:** Useful fleet-planning field with validation.

---

## 5.14 `Current Location`

**Data Dictionary meaning:** Most recent known vehicle location.

**Observed:**

- 12 encoded missing `Null`
- approximately 1,708 distinct non-missing strings

**Data type:** High-cardinality address text.

**Potential use:**

- GPS tracking diagnostics
- current-state visualization

**Major limitation:**

It is a snapshot associated with a Data Ping time and may not correspond to the historical planning cutoff required for backtesting.

**Recommended MVP action:**

Do not use it as the primary historical fleet-position source.

Prefer:

```text
completed trip -> destination zone
```

with explicit timing logic.

**Project role:** Secondary / diagnostic.

---

## 5.15 `Actual ETA`

**Data Dictionary meaning:** Actual arrival timestamp.

**Observed:**

- 25 encoded `NULL`
- 2 additional invalid 1899 time-only values
- valid operational dates roughly 2019-12-11 to 2020-08-29
- 7 valid comparable rows are earlier than Trip Start Date

**Primary use:**

- actual trip completion time
- delivery lateness
- vehicle availability after completed trips
- validation of `Ontime`

**Cleaning:**

```text
NULL -> missing
1899 -> invalid/missing
Actual ETA < Trip Start -> temporal anomaly flag
```

**Project role:** Essential for fleet-state reconstruction when valid.

**Forecast leakage:** Very high if used for a trip whose actual arrival had not yet occurred at the forecast cutoff.

---

## 5.16 `Current Location Latitude`

**Data Dictionary meaning:** Latitude of current GPS location.

**Observed:**

- 1 encoded missing value
- about 2,854 distinct values
- range approximately 8.701 to 32.368

**Potential use:**

- tracking visualization
- distance-to-destination
- GPS quality

**Caution:** Same historical-cutoff problem as Current Location.

**Project role:** Optional / diagnostic.

---

## 5.17 `Curren Location Longitude`

**Data Dictionary meaning:** Longitude of current GPS location.

**Schema issue:** The column name contains a typo: `Curren` instead of `Current`.

**Observed:**

- 1 encoded missing value
- about 2,829 distinct values
- range approximately 69.658 to 95.530

**Canonical rename:**

```text
current_location_longitude
```

**Use/caution:** Same as Current Location Latitude.

**Project role:** Optional / diagnostic.

---

## 5.18 `Ontime`

**Data Dictionary meaning:** Whether the vehicle arrived on time.

**Observed:**

- No: 2,204
- Yes: 1,381
- no missing values

**Validation finding:**

For valid comparable Planned ETA and Actual ETA rows, the label exactly matches:

```text
Yes if Actual ETA <= Planned ETA
No  if Actual ETA > Planned ETA
```

**Potential use:**

- delivery-performance EDA
- separate future project for ETA/on-time prediction

**Do not use for next-day demand forecasting.**

Reason:

It is determined after the trip outcome and is therefore a post-event feature.

**Project role:** Descriptive only for this project.

**Leakage risk:** Critical.

---

## 5.19 `Trip Start Date`

**Data Dictionary meaning:** Actual trip start date and time.

**Observed:**

- two invalid 1899 values
- valid operational range roughly 2019-11-29 to 2020-12-03
- about 3,074 distinct raw values

**This is the most important timestamp for demand construction.**

Primary use:

```text
departure_date = date(trip_start_datetime)
```

Target:

```text
count unique bookings by departure_date and origin_zone
```

**Cleaning:**

- convert 1899 values to missing
- require valid date for forecasting target
- preserve time for optional intraday analysis, but use daily granularity for MVP

**Project role:** Essential target-construction field.

---

## 5.20 `Trip End Date`

**Data Dictionary meaning in the supplied dictionary:** Estimated date and time of arrival.

**Important semantic concern:**

The column name sounds like actual trip end, but the dictionary describes it as an **estimated** arrival. This conflicts with the presence of both Planned ETA and Actual ETA.

**Observed:**

- two invalid 1899 values
- 57 comparable records occur earlier than Trip Start Date
- valid maximum around 2020-08-28

**Interpretation:** The semantics should be treated as uncertain.

**Recommended use:**

- inspect and compare against Data Ping time / Planned ETA / Actual ETA
- do not choose it as primary fleet-completion timestamp until meaning is proven

**Project role:** Quality investigation / possible fallback only.

---

## 5.21 `Transportation Distance (KM)`

**Data Dictionary meaning:** Total transportation distance in kilometers.

**Observed:**

- 148 encoded missing values: 4.13%
- 353 distinct non-missing values
- min: 0 km
- Q1: 107 km
- median: 400 km
- mean: ~841.1 km
- Q3: 1,290 km
- max: 2,898 km
- 18 zero-distance values

**Potential use:**

- network EDA
- historical zone characteristics
- identifying long-haul nature of the dataset
- validation against origin/destination geographic distance

**Forecasting use:**

Do not use tomorrow's known trip distance to predict tomorrow's trip count unless that shipment is already known at prediction time.

Safe historical feature:

```text
zone historical average distance
```

calculated only from past data.

**Project role:** Important EDA / historical feature.

---

## 5.22 `Vehicle Type`

**Data Dictionary meaning:** Type/configuration of vehicle.

**Observed:**

- 764 encoded `NULL`: 21.31%
- 38 observed non-missing categories

Top categories include:

- 32 FT Multi-Axle 14MT - HCV: 917
- 40 FT 3XL Trailer 35MT: 651
- 32 FT Single-Axle 7MT - HCV: 467
- 40 FT Flat Bed Multi-Axle 27MT - Trailer: 225
- 32 FT Multi-Axle MXL 18MT: 84

**Potential use:**

- advanced vehicle-class demand
- capacity/compatibility constraints
- fleet segmentation

**Issue:**

High missing rate + many categories will make zone × vehicle-type demand very sparse.

**Recommended MVP action:**

Ignore compatibility initially and solve a single-fleet rebalancing problem.

**Advanced action:**

Map 38 labels into broader groups such as:

```text
Light
Medium
Heavy
Trailer
Multi-Axle
Unknown
```

**Project role:** Advanced optimization feature.

---

## 5.23 `Minimum Kms To Be Covered In A Day`

**Data Dictionary meaning:** Minimum distance expected to be covered per day.

**Observed:**

- 2,645 missing: 73.78%
- only 940 non-missing
- only 2 non-missing unique values
- primarily 250 km, with some 275 km
- median 250 km

**Assessment:**

Very high missingness and extremely low information content.

**Potential use:**

Limited.

**Recommended MVP action:** Drop from forecasting and optimization.

Retain only in cleaned raw data for documentation.

**Project role:** Low value.

---

## 5.24 `Driver Name`

**Data Dictionary meaning:** Driver assigned to the trip.

**Observed:**

- 317 encoded `NA`: 8.84%
- approximately 1,239 distinct non-missing names

**Data type:** High-cardinality personal identifier.

**Potential use:** Almost none for this project.

**Privacy/modeling concern:**

Driver identity is not required for zone-demand forecasting or fleet repositioning and may introduce inappropriate person-specific correlations.

**Recommended action:**

- retain only in protected/raw layer if necessary
- exclude from model
- exclude from public dashboard/repository exports where possible

**Project role:** Exclude.

---

## 5.25 `Driver Mobile No`

**Data Dictionary meaning:** Driver contact number.

**Observed:**

- 1,024 missing: 28.56%
- approximately 1,192 distinct non-missing values

**Data type:** Identifier / contact information, often stored numerically in Excel.

**Critical note:**

Phone numbers should never be treated as numerical measurements.

**Potential use:** None for this project.

**Privacy concern:** Personally identifying/contact information.

**Recommended action:**

- remove from modeling tables
- do not publish
- do not visualize
- do not use for joins unless an explicit secure operational need exists

**Project role:** Exclude entirely from DS modeling.

---

## 5.26 `Customer Name`

**Data Dictionary meaning:** Customer receiving the shipment.

**Observed:**

- 36 distinct values
- no encoded missing values

Top customers include:

- Larsen & Toubro Limited: 977 raw rows
- Ford India Private Limited: 631
- Daimler India Commercial Vehicles Pvt Lt: 541
- Ashok Leyland Limited: 234
- Lucas Tvs Ltd: 209

**Potential use:**

- demand concentration EDA
- customer-origin relationship
- historical customer-mix features

**Forecast leakage warning:**

Tomorrow's customer identity cannot be used if it is not known before the prediction cutoff.

Safe alternative:

```text
historical zone customer concentration
lagged customer-share features
```

**Project role:** Useful EDA / optional historical feature.

---

## 5.27 `Supplier Name`

**Data Dictionary meaning:** Supplier sending the shipment.

**Observed:**

- 193 distinct raw non-missing values
- 192 after simple text normalization
- no encoded missing values
- `"Unknown"` itself appears frequently and should be considered a semantic missing/unknown category

Common values include:

- Ekta Transport Company
- Unknown
- Trans Cargo India
- Krc Logistics
- Vj Logistics

**Potential use:**

- supplier concentration EDA
- logistics-network analysis
- historical aggregate feature

**Cleaning:**

- normalize whitespace
- preserve `"Unknown"` as explicit unknown category or map to missing depending on analysis
- do not over-merge similarly named businesses without verification

**Project role:** Secondary.

**Leakage warning:** Do not use future trip supplier directly for next-day demand unless known at cutoff.

---

## 5.28 `Material Shipped`

**Data Dictionary meaning:** Description of transported material.

**Observed:**

- 712 distinct raw values
- 711 after basic text normalization
- high cardinality
- no encoded missing values

Common values include:

- Auto Parts: 1,212 raw rows
- Empty Trays: 231
- Grs Starter: 93
- Solenoid Assembly: 41
- Wiper Motor Link: 41

**Potential use:**

- shipment-composition EDA
- advanced material grouping
- understanding duplicated Booking IDs with multiple line items

**Important duplicate-booking implication:**

Different material descriptions can occur under the same physical Booking ID. Therefore Material Shipped helps explain why raw rows exceed unique trip count.

**Forecasting caution:**

Tomorrow's material cannot be used as a direct predictor unless known before cutoff.

**Recommended MVP action:**

- EDA only
- optionally create broad historical material groups later
- do not one-hot encode all 712 raw labels for the main demand model

**Project role:** Secondary / advanced.

---

# 6. Column-role matrix for this project

| Column | Cleaning | Demand target | Forecast feature | Fleet reconstruction | Optimization | EDA only / exclude |
|---|---|---|---|---|---|---|
| Gps Provider | Yes | No | No | Optional diagnostic | No | Mostly EDA |
| Booking ID | Yes | **Yes: unique count** | No | Join/key | No | Identifier |
| Shipment Type | Yes | No | Historical only | Optional | Advanced | Yes |
| Booking Date | Yes | No | Advanced only | Optional | No | Yes |
| Vehicle Registration | Yes | No | No | **Yes** | Supply identity | No |
| Origin Location | Yes | Label | No direct | Yes | Zone label | Yes |
| Destination Location | Yes | No | No future direct | **Yes** | Zone label | Yes |
| Origin Latitude | Yes | **Zone assignment** | Zone/static only | Yes | Yes | Yes |
| Origin Longitude | Yes | **Zone assignment** | Zone/static only | Yes | Yes | Yes |
| Destination Latitude | Yes | No | No future direct | **Yes** | Yes | Yes |
| Destination Longitude | Yes | No | No future direct | **Yes** | Yes | Yes |
| Data Ping time | Yes | No | No | Optional confidence | No | Yes |
| Planned ETA | Yes | No | No future direct | Optional | Availability | Yes |
| Current Location | Yes | No | No | Optional | No | Yes |
| Actual ETA | Yes | No | **Never future direct** | **Yes when known** | Availability | Yes |
| Current Location Latitude | Yes | No | No | Optional | No | Yes |
| Current Location Longitude | Yes | No | No | Optional | No | Yes |
| Ontime | Yes | No | **No: leakage** | No | No | Yes |
| Trip Start Date | **Yes** | **Essential** | Calendar/lags | Yes | Planning date | Yes |
| Trip End Date | Yes | No | No | Investigate | Optional | Yes |
| Transportation Distance | Yes | No | Historical only | Optional | Validation | Yes |
| Vehicle Type | Yes | No | Historical only | Advanced | Advanced | Yes |
| Minimum Kms/Day | Yes | No | Drop | No | No | Low value |
| Driver Name | Yes | No | Exclude | No | No | Exclude |
| Driver Mobile No | Yes | No | Exclude | No | No | Exclude |
| Customer Name | Yes | No | Historical only | No | No | Yes |
| Supplier Name | Yes | No | Historical only | No | No | Yes |
| Material Shipped | Yes | No | Historical only | No | Advanced | Yes |

---

# 7. Columns that should NOT go directly into the forecasting model

For the next-day demand model, exclude direct raw values of:

```text
Booking ID
Vehicle Registration
Driver Name
Driver Mobile No
Current Location
Current Location Latitude
Current Location Longitude
Actual ETA
Ontime
Trip End Date
future Destination Location
future Customer Name
future Supplier Name
future Material Shipped
future Vehicle Type
future Transportation Distance
```

Reason:

- identifiers
- personal information
- post-trip information
- information that may not exist at prediction time
- high-cardinality future attributes

---

# 8. Recommended canonical column names

| Raw column | Canonical name |
|---|---|
| Gps Provider | `gps_provider` |
| Booking ID | `booking_id` |
| Shipment Type | `shipment_type` |
| Booking Date | `booking_datetime` |
| Vehicle Registration | `vehicle_registration` |
| Origin Location | `origin_location` |
| Destination Location | `destination_location` |
| Origin Location Latitude | `origin_latitude` |
| Origin Location Longitude | `origin_longitude` |
| Destination Location Latitude | `destination_latitude` |
| Destination Location Longitude | `destination_longitude` |
| Data Ping time | `data_ping_datetime` |
| Planned ETA | `planned_eta` |
| Current Location | `current_location` |
| Actual ETA | `actual_eta` |
| Current Location Latitude | `current_latitude` |
| Curren Location Longitude | `current_longitude` |
| Ontime | `ontime` |
| Trip Start Date | `trip_start_datetime` |
| Trip End Date | `trip_end_datetime` |
| Transportation Distance (KM) | `transportation_distance_km` |
| Vehicle Type | `vehicle_type` |
| Minimum Kms To Be Covered In A Day | `minimum_km_per_day` |
| Driver Name | `driver_name` |
| Driver Mobile No | `driver_mobile_no` |
| Customer Name | `customer_name` |
| Supplier Name | `supplier_name` |
| Material Shipped | `material_shipped` |

---

# 9. Recommended derived variables

Do not modify raw columns; create derived fields.

## Data-quality fields

```text
is_valid_trip_start
is_valid_planned_eta
is_valid_actual_eta
is_valid_trip_end
booking_after_trip_start_flag
planned_eta_before_start_flag
actual_eta_before_start_flag
trip_end_before_start_flag
is_valid_distance
```

## Trip-level fields

```text
trip_date
trip_id
n_rows_per_booking
n_materials_per_booking
n_customers_per_booking
```

## Geospatial fields

```text
origin_point_id
destination_point_id
origin_zone
destination_zone
origin_zone_centroid_lat
origin_zone_centroid_lon
destination_zone_centroid_lat
destination_zone_centroid_lon
```

## Forecasting fields

```text
demand
demand_lag_1
demand_lag_2
demand_lag_3
demand_lag_7
demand_lag_14
rolling_mean_3
rolling_mean_7
rolling_mean_14
rolling_std_7
day_of_week
is_weekend
```

## Fleet fields

```text
vehicle_trip_count
is_recurrent_vehicle
estimated_vehicle_zone
available_flag
fleet_state_confidence
```

## Decision fields

```text
forecast_demand
available_supply
surplus
shortage
reposition_from_zone
reposition_to_zone
reposition_vehicle_count
reposition_distance_km
unmet_demand
```

---

# 10. Recommended first-pass treatment by column

## Keep and use heavily

```text
Booking ID
Vehicle Registration
Origin coordinates
Destination coordinates
Trip Start Date
Actual ETA
```

## Keep but validate carefully

```text
Booking Date
Data Ping time
Planned ETA
Trip End Date
Transportation Distance
Vehicle Type
```

## Keep mainly for EDA / historical aggregates

```text
Gps Provider
Shipment Type
Origin Location
Destination Location
Current Location
Ontime
Customer Name
Supplier Name
Material Shipped
```

## Drop from core model

```text
Minimum Kms To Be Covered In A Day
Driver Name
Driver Mobile No
```

## Avoid because of leakage

```text
Ontime
Actual ETA of future trips
future-trip destination/current-location/outcome data
```

---

# 11. Final data-model recommendation

The project should progressively transform the workbook into four analytical tables.

## Table A — `trip_level`

One row per unique Booking ID.

```text
booking_id
vehicle_registration
trip_start_datetime
origin_point
destination_point
planned_eta
actual_eta
distance
vehicle_type
...
```

## Table B — `daily_zone_demand`

One row per date × origin zone.

```text
date
origin_zone
demand
```

This is the forecasting table.

## Table C — `fleet_state_daily`

One row per planning date × vehicle.

```text
date
vehicle_registration
estimated_zone
available_flag
confidence
```

## Table D — `optimization_input`

One row per zone for a planning day.

```text
zone
forecast_demand
available_supply
surplus
shortage
```

Plus a separate zone-to-zone distance matrix.

This separation prevents row-level shipment fields from being accidentally used as future information in the forecasting model.

---

# 12. Most important conclusions

1. **Count unique bookings, not raw rows.**
2. **Use Trip Start Date as the demand timestamp.**
3. **Use coordinates, not raw location text, to build zones.**
4. **Use a dense core time period rather than assuming all missing historical dates equal zero demand.**
5. **Use recurrent vehicles for fleet-state inference instead of assuming all 1,262 vehicles are continuously observable.**
6. **Treat Actual ETA and Ontime as post-event data with strict leakage controls.**
7. **Do not use driver names or phone numbers in the model.**
8. **Do not start with 38 vehicle types; build a single-fleet MVP first.**
9. **Do not rely on Trip End Date until its semantics are resolved.**
10. **The dataset is best suited to a next-day logistics decision system, not long-term urban-mobility forecasting.**
