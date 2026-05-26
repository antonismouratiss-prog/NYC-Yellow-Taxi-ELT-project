# NYC Yellow Taxi — ELT Pipeline on Databricks
![Databricks](https://img.shields.io/badge/Databricks-FF3621?style=flat&logo=databricks&logoColor=white)
![Delta Lake](https://img.shields.io/badge/Delta_Lake-003366?style=flat&logo=delta&logoColor=white)
![PySpark](https://img.shields.io/badge/PySpark-E25A1C?style=flat&logo=apachespark&logoColor=white)
![Python](https://img.shields.io/badge/Python-3776AB?style=flat&logo=python&logoColor=white)

End-to-end ELT pipeline for NYC Yellow Taxi trip data built on Databricks using the Medallion Architecture (Bronze → Silver → Gold). The pipeline ingests raw trip records, applies data quality rules, and produces business-level aggregations using PySpark and Delta Lake.

---

## Architecture

```
TLC Public Source
      │
      ▼
┌─────────────┐
│   BRONZE    │  Raw ingestion — data landed as-is into Delta table
│             │  nyc_taxi.bronze.yellow_trips_raw
└──────┬──────┘
       │
       ▼
┌─────────────┐
│   SILVER    │  Cleaning, validation, data quality flagging
│             │  nyc_taxi.silver.yellow_trips_clean
└──────┬──────┘
       │
       ▼
┌─────────────┐
│    GOLD     │  Business-level aggregations
│             │  nyc_taxi.gold.*
└─────────────┘
```

---

## Dataset

| Property | Value |
|---|---|
| Source | [NYC TLC Trip Record Data](https://www.nyc.gov/site/tlc/about/tlc-trip-record-data.page) |
| Type | Yellow Taxi |
| Period | January 2024 |
| Raw row count | ~2.96M trips |
| Format | Parquet |

---

## Project Structure

```
nyc-taxi-ELT-databricks/
│
├── README.md
├── notebooks/
│   ├── 01_extract.ipynb
│   ├── 02_cleaning.ipynb
│   └── 03_aggregate.ipynb
└── images/
    └── architecture.png
```

---

## Notebooks

### 01 — Extract: Raw Ingestion
- Downloads the January 2024 Yellow Taxi Parquet file from the TLC public source
- Lands the file in a Unity Catalog Volume (`/Volumes/nyc_taxi/bronze/raw_files/`)
- Reads raw data with PySpark and saves it as-is to a Bronze Delta table
- No transformations applied — Bronze is the unmodified source of truth

### 02 — Cleaning: Validation & Quality
Applies the following data quality rules:

| Rule | Reason |
|---|---|
| Scope to January 2024 | Source file contains a small number of records outside the target window |
| `trip_distance > 0` | Zero distance trips are not valid passenger trips |
| `fare_amount > 0` | Zero or negative fares indicate invalid records |
| `trip_duration_minutes` between 0 and 300 | Removes impossible durations |
| `tpep_dropoff_datetime > tpep_pickup_datetime` | Ensures temporal consistency |
| Non-negative financial fields | Validates tip, tolls, surcharges and total amount |
| `PULocationID` and `DOLocationID` not null | Ensures location data is present |

Records with `passenger_count = 0` are not dropped but flagged using a `dq_flag` column:

| Flag | Meaning |
|---|---|
| `missing_passenger_count` | Zero passengers but valid duration and charge — likely a data entry gap |
| `no_passenger_no_charge` | Zero passengers and zero charge — ambiguous record |
| `invalid_trip` | Zero passengers and zero duration — likely a test or system record |
| `invalid_trip` | Positive passenger count but zero duration and zero charge |

This approach preserves full auditability while allowing downstream layers to filter by flag severity.

**Data quality result:** ~211K rows removed (~7.1% of raw data)

### 03 — Aggregate: Gold Layer
Three business-level aggregations built from the Silver clean table:

| Table | Description |
|---|---|
| `hourly_trip_volume` | Trip count by hour of day |
| `location_fare_stats` | Avg fare, tip and distance by pickup zone |
| `payment_type_distribution` | Trip count and avg tip by payment type |

---

## Key Findings

- **Peak demand** occurs at **18:00** with ~198K trips, reflecting NYC evening rush hour
- **Zone 43 (JFK Airport)** is a clear outlier — highest avg fare ($264.10) and avg tip ($58.75), consistent with long-distance airport transfers
- **Credit card** is the dominant payment method at **83.45%**, with ~17% of riders still paying cash — notably higher than most cities

---

## Tech Stack

| Tool | Purpose |
|---|---|
| Databricks (Community Edition) | Compute and notebook environment |
| PySpark | Distributed data processing |
| Delta Lake | ACID-compliant table format |
| Unity Catalog | Data governance and table management |
| Python | Orchestration and transformations |

---

## How to Run

1. Create a Databricks workspace with Unity Catalog enabled
2. Create catalog and schemas:
```sql
CREATE CATALOG nyc_taxi;
CREATE SCHEMA nyc_taxi.bronze;
CREATE SCHEMA nyc_taxi.silver;
CREATE SCHEMA nyc_taxi.gold;
CREATE VOLUME nyc_taxi.bronze.raw_files;
```
3. Run notebooks in order: `01_extract` → `02_cleaning` → `03_aggregate`
