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
  0_code/
    0_setup/
      catalog_setup
      data_upload        ← Kaggle → S3 ingestion notebook
    other/
      config             ← S3 paths and dataset configuration
      credentials        ← API keys (not pushed to GitHub)
  dictionary/
    data_dictionary      ← Column reference notebook
  LICENSE
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

### 🔄 Stage 2 — Bronze *(in progress)*
Raw files read from S3 and written to Delta tables in Databricks Unity Catalog with minimal transformation. Ingestion timestamp added to each record.

### ⬜ Stage 3 — Silver *(upcoming)*
Data cleaned, typed, deduplicated, and joined into a unified analytical layer.

### ⬜ Stage 4 — Gold *(upcoming)*
Aggregated tables built for the dashboard — survival rates by peak, season, weather conditions, and expedition type.

### ⬜ Stage 5 — Dashboard *(upcoming)*
Interactive visualisation and natural language querying via Databricks Dashboard and Genie.

---

## 💡 Key Design Decisions

### ELT over ETL
Data is loaded to S3 raw before any transformation occurs. This means raw data is always preserved and transformations can be rerun at any time without re-extracting from Kaggle. If transformation logic changes at any stage, the raw layer is untouched and acts as a reliable source of truth.

### IAM over Root
A dedicated IAM user was created for this project rather than using the AWS root account. Root credentials should never be used in application code. The IAM user has scoped S3 permissions, meaning if the credentials were ever compromised, the blast radius is limited to this project only.

### Encoding standardised at ingestion
`refer.csv` contained Latin-1 encoded characters causing codec errors on read. Rather than handling this in Silver or later, the re-encoding to UTF-8 happens immediately at ingestion before the file lands in S3. This means every layer downstream — Bronze, Silver, Gold — works with consistent UTF-8 data and encoding is never a concern again.

### Idempotent ingestion
The Kaggle to S3 ingestion notebook checks whether a file already exists in S3 before uploading. If it does, it is skipped. This means the notebook can be rerun at any time without duplicating data — important for a pipeline that may be triggered multiple times.

### Config separated from code
All dataset paths, Kaggle IDs, and S3 configuration live in a dedicated `config` notebook. This means the ingestion logic never needs to change if a path or source changes — only the config does. Credentials are stored separately and excluded from version control entirely.

---

## ⚙️ Setup

1. Clone the repo
2. Create a `credentials` notebook in `0_code/other/` with your Kaggle and AWS keys
3. Update `config` notebook with your S3 bucket name
4. Run `0_code/0_setup/data_upload` to ingest raw data into S3