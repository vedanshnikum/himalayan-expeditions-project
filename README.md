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
Silver (cleaned and joined)
    ↓
Gold (aggregated, dashboard-ready)
    ↓
Databricks Dashboard
```

---

## 📂 Repository Structure
```
himalayan-expeditions-project/
  0_scripts/
    0_setup/
      catalog_setup        ← Creates himalaya catalog and bronze/silver/gold schemas
      kaggle_to_s3         ← Pulls from Kaggle API and lands raw CSVs in S3
      s3_to_bronze         ← Reads from S3 and writes Delta tables to himalaya.bronze
    configs/
      config               ← S3 paths, dataset config, S3 client
      credentials          ← API keys (not pushed to GitHub)
  dictionary/
    himalayan_data_dictionary.csv
    data_dictionary        ← Renders data dictionary as interactive table
  README.md
  .gitignore
```

---

## 📊 Data Sources

| Dataset | Author | Last Updated |
|---|---|---|
| [Mountain Climbing Accidents Dataset](https://www.kaggle.com/datasets/asaniczka/mountain-climbing-accidents-dataset) | asaniczka | 2 years ago |
| [Historic Weather Data for Himalayan Peaks](https://www.kaggle.com/datasets/bonesclarke26/historic-weather-data-for-himalayan-peaks) | Ryan Clarke | 7 months ago |
| [Himalayan Expeditions](https://www.kaggle.com/datasets/siddharth0935/himalayan-expeditions) | Siddharth Vora | 9 months ago |
| [Mount Everest Accident Dataset 2020–2025](https://www.kaggle.com/datasets/syedmuhammadbilal12/mount-everest-accident-dataset-2020-2025) | Syed Muhammad Bilal | 2 months ago |

---

## 🔄 Pipeline Stages

### ✅ Stage 1 — Ingestion (Kaggle → S3)
Raw datasets pulled from Kaggle using `kagglehub` and written to AWS S3 as UTF-8 encoded CSV files. Encoding inconsistencies handled at ingestion. Idempotent — skips files that already exist in S3.

### ✅ Stage 2 — Bronze
Raw files read from S3 and written as Delta tables to `himalaya.bronze` in Databricks Unity Catalog. Column names standardised to lowercase with underscores. Ingestion timestamp added to each record. Idempotent — skips tables that already exist.

### ⬜ Stage 3 — Silver *(upcoming)*
Data cleaned, typed, deduplicated, and joined into a unified analytical layer.

### ⬜ Stage 4 — Gold *(upcoming)*
Aggregated tables built for the dashboard — survival rates by peak, season, weather conditions, and expedition type.

### ⬜ Stage 5 — Dashboard *(upcoming)*
Interactive visualisation and natural language querying via Databricks Dashboard and Genie.

---

## 💡 Key Design Decisions

### ELT over ETL
Data is loaded to S3 raw before any transformation occurs. Raw data is always preserved and transformations can be rerun at any time without re-extracting from Kaggle. If transformation logic changes at any stage, the raw layer is untouched and acts as a reliable source of truth.

### IAM over Root
A dedicated IAM user was created for this project rather than using the AWS root account. Root credentials should never be used in application code. The IAM user has scoped S3 permissions, meaning if credentials were ever compromised, the blast radius is limited to this project only.

### Encoding standardised at ingestion
`refer.csv` contained Latin-1 encoded characters causing codec errors on read. Rather than handling this in Silver or later, re-encoding to UTF-8 happens immediately at ingestion before the file lands in S3. Every layer downstream works with consistent UTF-8 data.

### Column names standardised at Bronze
Source data contained column names with spaces and special characters which Delta Lake does not support. All column names are lowercased and non-alphanumeric characters replaced with underscores at Bronze ingestion, ensuring consistent naming across all downstream layers.

### Idempotent ingestion
Both the Kaggle → S3 and S3 → Bronze notebooks check whether data already exists before writing. This means both notebooks can be rerun safely at any time without duplicating data.

### Config separated from code
All dataset paths, Kaggle IDs, and S3 configuration live in a dedicated `config` notebook. Ingestion logic never needs to change if a path or source changes — only the config does. Credentials are stored separately and excluded from version control entirely.

---

## ⚙️ Setup

1. Clone the repo
2. Create a `credentials` notebook in `0_scripts/configs/` with your Kaggle and AWS keys
3. Update `config` notebook with your S3 bucket name
4. Run `0_scripts/0_setup/catalog_setup` to create the Databricks catalog and schemas
5. Run `0_scripts/0_setup/kaggle_to_s3` to ingest raw data into S3
6. Run `0_scripts/0_setup/s3_to_bronze` to load Bronze Delta tables
