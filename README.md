# 🏔️ Himalayan Expeditions Data Pipeline

## Overview

This project explores how weather conditions have historically influenced climbing outcomes in the Himalayas. Using decades of expedition records, fatality data, and weather data across major Himalayan peaks, the pipeline analyzes patterns between conditions and climbing success or failure.

Incremental weather data is simulated by uploading historic weather files in batches — mimicking how a real pipeline would ingest newly available data over time. Comparisons are made against historical baselines to assess how dangerous conditions were during past climbs.

> **Note on predictions:** No machine learning is used. Analysis is pattern-based comparison against historic data only.

> **Note on incremental data:** As live Himalayan weather APIs are not publicly available, historic weather data past a cutoff date is uploaded incrementally to simulate a real-time ingestion pattern.

---

## 🛠️ Tech Stack

| Tool | Purpose |
|---|---|
| Python / PySpark | Pipeline logic and data transformation |
| SQL | Querying and table creation |
| AWS S3 | Raw data lake and object storage |
| Databricks Community Edition | ELT processing and dashboarding |
| Delta Lake | ACID-compliant table storage |
| Databricks Workflows | Pipeline orchestration |
| Databricks Dashboard | BI reporting and visualisation |
| Databricks Genie | Natural language querying |

---

## 🏗️ Architecture

This project follows a **Data Lakehouse** pattern with **Medallion Architecture** (Bronze → Silver → Gold).

```
Kaggle API
    ↓
AWS S3 (raw layer)
    ↓
Bronze (raw Delta tables)
    ↓
Silver (cleaned and transformed)
    ↓
Gold (aggregated, dashboard-ready)
    ↓
Databricks Dashboard
```

---

## 📂 Repository Structure

```
himalayan-expeditions-project/
  scripts/
    0_setup/
      catalog_setup              ← Creates himalaya catalog and bronze/silver/gold schemas
      kaggle_to_s3               ← Pulls from Kaggle API and lands raw CSVs in S3
    1_bronze/
      full_load/
        s3_to_bronze             ← Reads from S3 and writes Delta tables to himalaya.bronze
      incremental/
        weather_to_bronze        ← Reads parquet from S3 landing, appends to himalaya.bronze.weather
    2_silver/
      full_load/
        silver_deaths            ← Cleans and transforms deaths table
        silver_expeditions_exped ← Cleans and transforms expeditions table
        silver_expeditions_members ← Cleans and transforms members table
        silver_expeditions_peaks ← Cleans and transforms peaks table
      incremental/
        silver_weather           ← Cleans and merges weather data into himalaya.silver.weather
    3_gold/
      gold_dim_peaks             ← Peak reference dimension
      gold_dim_weather           ← Daily weather dimension (incremental)
      gold_dim_members           ← Climber dimension
      gold_dim_deaths            ← Deaths dimension
      gold_fact_expeditions      ← Core fact table, one row per expedition
    exploration/
      explore_deaths
      explore_expeditions_exped
      explore_expeditions_members
      explore_expeditions_peaks
      explore_weather
  dashboarding/
    create_views                 ← SQL views for dashboard consumption
  configs/
    config                       ← S3 paths, dataset config
    credentials                  ← API keys (not pushed to GitHub)
  references/
    data_dictionary              ← Column reference for all tables
    refer                        ← Source citations and bibliography
  README.md
  LICENSE
  .gitignore
```

---

## 📊 Data Sources

| Dataset | Author | Last Updated |
|---|---|---|
| [Mountain Climbing Accidents Dataset](https://www.kaggle.com/datasets/asaniczka/mountain-climbing-accidents-dataset) | asaniczka | 2024 |
| [Historic Weather Data for Himalayan Peaks](https://www.kaggle.com/datasets/bonesclarke26/historic-weather-data-for-himalayan-peaks) | Ryan Clarke | 2025 |
| [Himalayan Expeditions](https://www.kaggle.com/datasets/siddharth0935/himalayan-expeditions) | Siddharth Vora | 2025 |

---

## 🔄 Pipeline Stages

### ✅ Stage 1 — Ingestion (Kaggle → S3)
Raw datasets pulled from Kaggle using `kagglehub` and written to AWS S3 as UTF-8 encoded CSV files. Encoding inconsistencies handled at ingestion. Idempotent — skips files that already exist in S3.

### ✅ Stage 2 — Bronze
Raw files read from S3 and written as Delta tables to `himalaya.bronze` in Databricks Unity Catalog. Column names standardised to lowercase with underscores. Ingestion timestamp added to each record. Full load is idempotent — skips tables that already exist. Incremental weather data is appended on each run with a staging table written for downstream Silver processing.

### ✅ Stage 3 — Silver
Data cleaned, typed, and transformed per table. Irrelevant columns dropped, date columns cast to DateType, categorical columns consolidated and standardised, column names cleaned and renamed. Weather Silver uses Delta MERGE to handle incremental updates without duplicates.

| Table | Key Transformations |
|---|---|
| `silver_deaths` | Date cast, cause of death consolidated into categories, nationality dropped |
| `silver_expeditions_exped` | 30+ columns dropped, routes/successes/deaths consolidated, dates cast, columns renamed |
| `silver_expeditions_members` | 40+ columns dropped, name consolidated, dates cast, nationality cleaned, columns renamed |
| `silver_expeditions_peaks` | 11 columns dropped, types cast, columns renamed |
| `silver_weather` | 9 columns dropped, hourly aggregated to daily, date cast, columns renamed, incremental merge |

### ✅ Stage 4 — Gold
Snowflake schema with one fact table and four dimension tables. All tables sourced from Silver. Weather dimension is updated incrementally as new peak batches arrive.

| Table | Description |
|---|---|
| `fact_expeditions` | One row per expedition — outcomes, metrics, weather_id key |
| `dim_peaks` | Peak reference data — height, region, first ascent details |
| `dim_weather` | Daily weather per peak — temperature, wind, snowfall, conditions |
| `dim_members` | One row per climber — nationality, role, success, death boolean |
| `dim_deaths` | Death records enriched with peak and expedition context |

### ✅ Stage 5 — Dashboarding
Interactive visualisation via Databricks Dashboard and natural language querying via Databricks Genie. SQL views built in `dashboarding/create_views` serve as the consumption layer for all dashboard pages.

The dashboard is split into six pages, each surfacing a different analytical lens:

**Overview**
High-level summary of the full dataset — total expeditions, deaths, mountains, members, average success rate, and year range. Includes trend lines for expeditions by year, success vs failure by year, most climbed peaks, expeditions by season, and a combined expeditions vs deaths chart over time.

**Mountain Info**
Per-mountain breakdowns including the highest, hardest (lowest success rate among summited peaks), most climbed, and deadliest mountains. Charts cover average success rate by mountain, average death rate by mountain, average oxygen usage by mountain, expeditions per mountain, and regional distribution of peaks across the Himalayas.

**Member Info**
Climber-level analysis including a choropleth map of member nationalities, sex breakdown, youngest and oldest climbers (name, nationality, age), and leader age comparison against non-leaders.

**Expedition Analysis**
Deeper expedition-level exploration including termination reasons, success rate by season, success rate with vs without hired staff, success rate with vs without oxygen, expedition counts by season, most expeditions by an individual, and the most leaders recorded in a single expedition.

**Death Analysis**
Fatality analysis including total deaths, deadliest mountain, deaths per year, deaths by oxygen status, deaths by Sherpa status, most deaths in a single expedition, and a Sankey diagram showing how deaths break down by oxygen/hired status through to cause of death.

**Weather Analysis**
Weather impact on climbing outcomes — most common weather condition, average temperature and wind speed across all records, success rate by weather condition, deaths by weather condition, average temperature and wind speed trends by year, scatter plots of wind speed and temperature against highest point reached, and snowfall vs success rate.

---

## ⚙️ Orchestration

The pipeline is orchestrated as a single Databricks Workflow with 15 tasks. The full load path and incremental weather path run in parallel from `catalog_setup`, merging back at the Gold layer before `create_views` runs at the end.

```
catalog_setup
    ├── kaggle_to_s3
    │       └── s3_to_bronze
    │               ├── silver_expeditions_exped  ──┐
    │               ├── silver_deaths             ──┤
    │               ├── silver_expeditions_members──┼──► gold (x5) ──► create_views
    │               └── silver_expeditions_peaks  ──┘        ▲
    └── weather_to_bronze                                     │
                └── silver_weather ──────────────────── gold_dim_weather
```

| Task | Notebook | Depends On |
|---|---|---|
| 1 | `catalog_setup` | — |
| 2 | `kaggle_to_s3` | catalog_setup |
| 3 | `s3_to_bronze` | kaggle_to_s3 |
| 4 | `weather_to_bronze` | catalog_setup |
| 5 | `silver_expeditions_exped` | s3_to_bronze |
| 6 | `silver_deaths` | s3_to_bronze |
| 7 | `silver_expeditions_members` | s3_to_bronze |
| 8 | `silver_expeditions_peaks` | s3_to_bronze |
| 9 | `silver_weather` | weather_to_bronze |
| 10 | `gold_fact_expeditions` | tasks 5,6,7,8,9 |
| 11 | `gold_dim_deaths` | tasks 5,6,7,8,9 |
| 12 | `gold_dim_members` | tasks 5,6,7,8,9 |
| 13 | `gold_dim_peaks` | tasks 5,6,7,8,9 |
| 14 | `gold_dim_weather` | tasks 5,6,7,8,9 |
| 15 | `create_views` | tasks 10,11,12,13,14 |

---

## 💡 Key Design Decisions

### ELT over ETL
Data is loaded to S3 raw before any transformation occurs. Raw data is always preserved and transformations can be rerun at any time without re-extracting from Kaggle. If transformation logic changes at any stage, the raw layer is untouched and acts as a reliable source of truth.

### IAM over Root
A dedicated IAM user was created for this project rather than using the AWS root account. Root credentials should never be used in application code. The IAM user has scoped S3 permissions, meaning if credentials were ever compromised, the blast radius is limited to this project only.

### Full load vs incremental ingestion
Expedition and deaths datasets are static historical records loaded once from Kaggle. Weather data is ingested incrementally — new peak files are uploaded to S3 landing in batches, processed into Bronze, and merged into Silver. This simulates a real-world pipeline where new data arrives over time.

### Single workflow with parallel branches
Rather than maintaining two separate workflows, the full load and incremental weather paths are combined into one Databricks Workflow. Both branches originate from `catalog_setup` and converge at the Gold layer, giving a single view of the entire pipeline's run history and making all task dependencies explicit.

### Snowflake schema in Gold
Gold follows a snowflake schema with `fact_expeditions` at the center referencing four dimension tables. `dim_deaths` is a child dimension of `dim_members`, linking via name and peakid. This structure supports flexible dashboard queries while maintaining clear separation of concerns between expedition outcomes, climber details, peak metadata, and weather conditions.

### Staging tables for incremental processing
Each incremental weather batch is written to a staging table in Bronze before Silver processing. Silver reads only the staging table rather than the full Bronze weather table, ensuring efficient processing regardless of how large the historical weather data grows.

### Encoding standardised at ingestion
Source files contained Latin-1 encoded characters causing codec errors on read. Rather than handling this in Silver or later, re-encoding to UTF-8 happens immediately at ingestion before files land in S3. Every layer downstream works with consistent UTF-8 data.

### Column names standardised at Bronze
Source data contained column names with spaces and special characters which Delta Lake does not support. All column names are lowercased and non-alphanumeric characters replaced with underscores at Bronze ingestion, ensuring consistent naming across all downstream layers.

### Idempotent ingestion
Both the Kaggle → S3 and S3 → Bronze notebooks check whether data already exists before writing. This means both notebooks can be rerun safely at any time without duplicating data.

### Transformations deferred to Silver
Raw data is never modified at Bronze. All cleaning, consolidation, and restructuring happens in Silver. This means Bronze always reflects the source data exactly, and Silver transformations can be rerun or modified without re-ingesting from S3.

### Reference data excluded from pipeline
The expedition references table contains bibliographic citations with no analytical value. It is excluded from the pipeline and stored separately in the `references/` folder alongside the data dictionary for documentation purposes only.

### Config separated from code
All dataset paths, Kaggle IDs, and S3 configuration live in a dedicated `config` notebook. Ingestion logic never needs to change if a path or source changes — only the config does. Credentials are stored separately and excluded from version control entirely.

---

## 🚀 Running the Pipeline

Before the first run:

1. Clone the repo into your Databricks workspace
2. Create a `credentials` notebook in `configs/` with your Kaggle and AWS keys
3. Update the `config` notebook with your S3 bucket name
4. Upload weather parquet files to `s3://himalaya-dp-vn/raw/incremental_load/historic-weather-data/landing/`
5. Trigger the workflow from the Databricks Workflows UI

All 15 tasks execute in dependency order. The full load and incremental weather branches run in parallel, converging at Gold before `create_views` finalises the dashboard consumption layer.
