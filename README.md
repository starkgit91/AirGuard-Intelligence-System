# AirGuard-Intelligence-System
A district-level triage system that fuses AQI, healthcare deficits, industrial hazard maps &amp; vulnerable population data into a single 0–100 risk score — built on Databricks Lakehouse.

# AirGuard — AI-Powered Pollution Impact & City Preparedness Intelligence System

> **What it does:** AirGuard fuses CPCB air quality data with NHP health infrastructure data to compute a city-level Preparedness Score — identifying which Indian cities are most at risk from pollution *and* least equipped to handle it. Mosaic AI (Llama 3.3 70B) generates actionable policy recommendations per city.

Built end-to-end on **Databricks** using Delta Lake Medallion Architecture, Unity Catalog, MLflow, Mosaic AI Foundation Models, and Databricks Apps.

---

## Architecture Diagram

```
┌──────────────────────────────────────────────────────────────────┐
│                          DATA SOURCES                            │
│   CPCB AQI (city_day.csv) · NHP Health Infra · Census           │
└──────────────┬───────────────────────────────┬───────────────────┘
               │                               │
               ▼                               ▼
┌──────────────────────────────────────────────────────────────────┐
│  BRONZE LAYER — Unity Catalog · Delta Lake                       │
│  workspace.bronze.aqi_raw  |  workspace.bronze.health_raw        │
│  (raw CSVs ingested from /Volumes/workspace/bronze/raw_data/)    │
└──────────────────────────┬───────────────────────────────────────┘
                           │  PySpark cleaning · null imputation
                           │  city name standardisation
                           ▼
┌──────────────────────────────────────────────────────────────────┐
│  SILVER LAYER — Cleaned · Joined · Normalised                    │
│  workspace.silver.city_master                                    │
│  (1 row per city — AQI summary + health infra aggregated)        │
└──────────────────────────┬───────────────────────────────────────┘
                           │  Preparedness Score formula
                           │  XGBoost AQI forecast
                           ▼
┌──────────────────────────────────────────────────────────────────┐
│  GOLD LAYER — Preparedness Score + Forecast + LLM Recs           │
│  workspace.gold.city_preparedness  (MLflow tracked, versioned)   │
│  workspace.gold.aqi_forecast                                     │
└──────────┬────────────────────────────────────┬──────────────────┘
           │ Mosaic AI Foundation Models         │ Databricks Apps
           │ Llama 3.3 70B Instruct              │ (Gradio)
           ▼                                     ▼
  Per-city policy recommendations        Interactive Dashboard
  cached in Gold layer                   City rankings · Forecast
                                         LLM drilldown · MLflow logs
```

**Preparedness Score Formula:**
```
Pollution Impact Index  = 0.6 × AQI_norm + 0.4 × PM2.5_norm
Health Infra Index      = 0.4 × hospitals_norm + 0.3 × facilities_norm
                        + 0.2 × PHC_norm + 0.1 × CHC_norm
Preparedness Score      = normalise(Health Infra Index − Pollution Impact Index)
```
Score >= 0.65 → Well Prepared | 0.35–0.65 → Moderate Risk | < 0.35 → Critically Vulnerable

---

## Databricks Technologies Used

| Technology | Usage |
|---|---|
| **Delta Lake** | Bronze/Silver/Gold tables with ACID transactions and time-travel |
| **Unity Catalog** | Governed table storage, lineage, and access control |
| **Apache Spark (PySpark)** | Full data cleaning, join, and normalisation pipeline |
| **MLflow** | Experiment tracking for preparedness scoring and XGBoost forecasting |
| **Mosaic AI Foundation Models** | Llama 3.3 70B Instruct for per-city policy recommendations |
| **Databricks Apps** | Gradio dashboard deployment |
| **Databricks Workflows** | Orchestrated pipeline (pipeline_workflow.json) |
| **Databricks Volumes** | Raw CSV file storage under Unity Catalog |

---

## Project Write-up (500 characters)

AirGuard fuses CPCB air quality data with NHP health infrastructure to compute a city-level Preparedness Score across 26 Indian cities using Databricks Medallion Architecture (Bronze to Silver to Gold on Delta Lake + Unity Catalog). XGBoost forecasts 3-day AQI with MLflow tracking. Mosaic AI (Llama 3.3 70B) auto-generates actionable policy recommendations per city. A Gradio dashboard deployed on Databricks Apps makes results accessible to policymakers.

---

## Accuracy Metrics

| Model | Metric | Value |
|---|---|---|
| XGBoost AQI Forecaster | MAE | 14.16 AQI points |
| XGBoost AQI Forecaster | RMSE | 30.86 AQI points |
| Preparedness Score | Cities scored | 26 |
| Preparedness Score | Critically Vulnerable | 7 |
| Preparedness Score | Moderate Risk | 8 |
| Preparedness Score | Well Prepared | 11 |

---

## How to Run

### Prerequisites
1. Databricks workspace with Unity Catalog enabled
2. Cluster: DBR 15.4 LTS ML or above (ML Runtime required)
3. Upload data files to Unity Catalog Volume (see File Structure below)

### File Structure — Upload to Databricks Volume

```
/Volumes/workspace/bronze/raw_data/
├── aqi/
│   └── city_day.csv
├── health/
│   └── geocode_health_centre.csv
└── census/
    └── populaion_testing - Sheet1.csv
```

Upload command:
```bash
databricks fs cp data/ /Volumes/workspace/bronze/raw_data/ --recursive
```

### Step 1 — Setup Unity Catalog schemas

Run in a Databricks notebook cell:
```python
spark.sql("CREATE SCHEMA IF NOT EXISTS workspace.silver")
spark.sql("CREATE SCHEMA IF NOT EXISTS workspace.gold")
```

### Step 2 — Run the Main Pipeline Notebook

```bash
databricks workspace import project.ipynb /Workspace/airguard/project --format JUPYTER
```

Then open in Databricks, attach to a DBR 15.4 LTS ML cluster, and click Run All.

Or trigger via Databricks Workflows:
```bash
databricks jobs create --json @pipeline_workflow.json
databricks jobs run-now --job-id <job-id>
```

### Step 3 — Deploy the Gradio Dashboard

```bash
databricks apps create airguard-app
databricks apps deploy airguard-app --source-code-path /Workspace/airguard/app
```

Expected runtime: 8–12 minutes (full pipeline)

---

## Demo Steps

1. Open the deployed Databricks App URL (from `databricks apps list`)

2. City Profile tab — select any city from the dropdown
   - See: Preparedness Score, tier, Pollution Index vs Health Infra Index
   - See: Mosaic AI policy recommendation (Llama 3.3 70B)
   - Try: Switch between "Gurugram" (Critically Vulnerable) and "Ernakulam" (Well Prepared)

3. 3-Day Forecast tab — select "Delhi" or "Ahmedabad"
   - See: AQI forecast for Day+1, Day+2, Day+3 with risk categories and dates

4. In the notebook — scroll to Cell 8 (Gold Layer)
   - See: MLflow experiment run logged with preparedness metrics
   - Open Databricks sidebar → Experiments → /Shared/airguard/scoring-pipeline

5. In Unity Catalog — open Catalog Explorer
   - Navigate to: workspace → gold → city_preparedness
   - See: Delta table with lineage, schema, and table comment

6. Query the Gold table directly:
```python
display(spark.table("workspace.gold.city_preparedness").orderBy("preparedness_score"))
```

---

## Repository Structure

```
airguard/
├── project.ipynb              ← Main pipeline notebook (run this)
├── app/
│   └── app.py                 ← Gradio dashboard (Databricks Apps)
├── pipeline_workflow.json     ← Databricks Workflows job definition
├── setup_catalog.py           ← Unity Catalog schema setup
├── requirements.txt           ← Python dependencies
└── README.md                  ← This file
```

---

## Key Results

| Rank | City | Avg AQI | Preparedness Score | Tier |
|---|---|---|---|---|
| 1 (worst) | Gurugram | 222.4 | 0.000 | Critically Vulnerable |
| 2 | Patna | 231.1 | 0.043 | Critically Vulnerable |
| 3 | Ahmedabad | 401.8 | 0.046 | Critically Vulnerable |
| 25 | Thiruvananthapuram | 74.2 | 0.989 | Well Prepared |
| 26 (best) | Ernakulam | 90.6 | 1.000 | Well Prepared |

---

## Requirements

```
gradio>=4.20.0
plotly>=5.18.0
folium>=0.15.0
pandas>=2.0.0
databricks-connect>=14.0.0
mlflow>=2.10.0
databricks-sdk>=0.20.0
xgboost>=1.7.0
scikit-learn>=1.3.0
```

---

## Team

Gaurav — PhD Research Scholar, Machine Learning, IIT Indore

*Built for Databricks Hackathon 2026*
