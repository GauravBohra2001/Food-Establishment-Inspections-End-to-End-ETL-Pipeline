# Food Inspection Data Pipeline
A end-to-end Medallion Architecture pipeline built on **Databricks** and **Delta Tables** that ingests food inspection data from Dallas and Chicago public APIs, transforms it through Bronze, Silver, and Gold layers, and models it into a Kimball-style star schema for analytics.

---

## Overview

This project builds a production-style data engineering pipeline as part of a graduate-level Data Architecture course at Northeastern University. The pipeline ingests raw food inspection records from two city open data APIs, standardizes and cleanses the data, and loads it into a dimensional model optimized for reporting and analysis.

**Key goals:**
- Demonstrate a metadata-driven, reusable ingestion framework
- Apply Medallion Architecture principles (Bronze → Silver → Gold)
- Implement SCD Type 2 for restaurant dimension tracking
- Enforce data quality with row-count validation and quarantine logging

---

## Architecture

```
Open Data APIs
  Dallas Open Data  ──┐
  Chicago Data Portal ──┼──► Bronze Layer ──► Silver Layer ──► Gold Layer
                        │    (Raw Delta)      (Cleansed)       (Star Schema)
                   Metadata-driven
                   ingestion loop
```

### Layer Breakdown

| Layer | Description |
|---|---|
| **Bronze** | Raw data landed as-is from API. Parquet written to staging volume, then loaded to Delta tables with batch metadata columns. |
| **Silver** | Cleansed, standardized, and unioned Dallas + Chicago records into a single conformed table. |
| **Gold** | Kimball star schema — `fact_inspections`, `fact_violations`, `dim_restaurant` (SCD Type 2), `dim_violation_detail`, `dim_date`. Quarantine tables capture rejected records. |

### Metadata Tables

- **`parent_metadata`** — controls which tables are active for ingestion
- **`child_metadata`** — logs every pipeline run with status, row counts, file path, and timestamp

---

## Data Sources

| City | API Endpoint | Rows |
|---|---|---|
| Dallas | `dallasopendata.com/resource/dri5-wcct.csv` | ~79,000 |
| Chicago | `data.cityofchicago.org/resource/4ijn-s7e5.csv` | ~100,000 |

---

## Tech Stack

- **Platform:** Databricks
- **Storage:** Unity Catalog Volumes + Delta Tables
- **Language:** PySpark (Python)
- **Modeling:** Star Schema Dimensional Modeling
- **Orchestration:** Notebook-based pipeline with metadata-driven control

---

## Setup & How to Run

### Prerequisites
- Databricks workspace with Unity Catalog enabled
- A cluster with access to external URLs (for API downloads)

### 1. Configure Utils
Open `utils.py` and verify the constants match your workspace:
```python
CATALOG       = "Final_Project"
BRONZE_SCHEMA = "Bronze_Zone"
SILVER_SCHEMA = "Silver_Zone"
GOLD_SCHEMA   = "Gold_Zone"
META_SCHEMA   = "pipeline_metadata"
```

### 2. Run Environment Setup
Run the environment setup notebook to create the catalog, schemas, volumes, and metadata tables:
```
notebooks/00_env_setup.py
```

### 3. Seed Parent Metadata
Insert active table entries so the pipeline knows what to process:
```sql
INSERT INTO Final_Project.pipeline_metadata.parent_metadata VALUES
  ('Dallas',  'Dallas_Restaurant_and_Food_Establishment_Inspections.csv',  'Y', current_date(), current_date()),
  ('Chicago', 'Chicago_Restaurant_and_Food_Establishment_Inspections.csv', 'Y', current_date(), current_date());
```

### 4. Run the Pipeline
Execute notebooks in order:
```
notebooks/01_raw_to_bronze.py   ← Downloads from API, writes to Bronze Delta tables
notebooks/02_bronze_to_silver.py ← Cleanses and unions both cities
notebooks/03_silver_to_gold.py   ← Loads star schema dimensions and fact tables
```

### 5. Monitor Runs
Query the child metadata table to check pipeline status:
```sql
SELECT * FROM Final_Project.pipeline_metadata.child_metadata
ORDER BY execution_time DESC;
```

---

## Project Structure

```
├── utils.py                   # All constants and path variables
├── notebooks/
│   ├── 00_env_setup.py        # Catalog, schema, volume, metadata table creation
│   ├── 00_gold_setup.py       # Gold layer table DDL
│   ├── 01_raw_to_bronze.py    # API download → staging parquet → Bronze Delta
│   ├── 02_bronze_to_silver.py # Cleanse, standardize, union → Silver
│   └── 03_silver_to_gold.py   # Dimensional model → Gold
└── README.md
```
