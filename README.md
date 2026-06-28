[README.md](https://github.com/user-attachments/files/29441154/README.md)
# IT Infrastructure Anomaly Detection — End-to-End MLOps Pipeline

> Predicting server failures before they happen using real-time telemetry data, Databricks, XGBoost, and a fully automated MLOps lifecycle.

---

## Overview

This project builds a production-grade MLOps pipeline for detecting anomalies in server infrastructure telemetry. It takes multivariate time-series data from 28 servers across 38 metrics (CPU, memory, disk I/O, network) and predicts anomalies in real time — before they escalate into incidents.

The pipeline covers every stage of the ML lifecycle: data ingestion, feature engineering, model training, experiment tracking, model registry, real-time serving, and drift monitoring.

**Key results:**
- AUC: **0.992**
- Recall: **95%** (catches 95 out of every 100 real anomalies)
- Dataset: **708,405 rows** across 28 machines
- Anomaly rate: **4.2%** (realistic production imbalance)

---

## Architecture

```
SMD Dataset (38 metrics · 28 servers · .txt files)
        │
        ▼
┌─────────────────────────────────────────────────────┐
│                Medallion Architecture                │
│                                                     │
│  Bronze Layer          Silver Layer     Gold Layer  │
│  ─────────────  ──►  ────────────── ──► ─────────  │
│  Raw ingestion         Clean + rolling   44 features │
│  Delta Lake            window features   model-ready │
└─────────────────────────────────────────────────────┘
        │
        ▼
┌─────────────────────────────────────────────────────┐
│              XGBoost Classifier                     │
│  AUC 0.992 · Recall 95% · scale_pos_weight=23      │
└─────────────────────────────────────────────────────┘
        │                        │
        ▼                        ▼
┌──────────────┐      ┌──────────────────────┐
│    MLflow    │      │    Model Registry    │
│  Experiment  │      │  Champion v1         │
│  tracking    │      │  Versioned + aliased │
└──────────────┘      └──────────────────────┘
                               │
                               ▼
                  ┌────────────────────────┐
                  │     Model Serving      │
                  │  Live REST endpoint    │
                  │  Real-time scoring     │
                  └────────────────────────┘
                               │
                               ▼
                  ┌────────────────────────┐
                  │  Evidently Monitoring  │
                  │  Weekly drift report   │
                  │  Alert if >30% drift   │
                  └────────────────────────┘
```

---

## Dataset

**Server Machine Dataset (SMD)**
- Source: [NetManAIOps/OmniAnomaly](https://github.com/NetManAIOps/OmniAnomaly)
- 28 server machines across 3 groups
- 38 metrics per machine (CPU, memory, disk read/write, network send/receive)
- Labelled anomalies in the test set
- Train: 708,405 rows · Test: 708,420 rows · Anomaly rate: 4.2%

---

## Pipeline breakdown

### Bronze layer — raw ingestion
- Reads all 28 server `.txt` files from Databricks Unity Catalog Volume
- Assigns column names (`metric_0` to `metric_37`) and tags each row with its source machine
- Writes to Delta Lake Bronze table with ACID compliance

### Silver layer — cleaning and feature engineering
- Fills nulls (metrics are normalised 0–1, so 0 is a safe default)
- Adds a per-machine `time_step` index to preserve temporal ordering
- Engineers rolling window features per machine:

| Feature | Window | What it captures |
|---|---|---|
| `metric_0_mean_5` | 5 steps | Short-term trend |
| `metric_0_mean_15` | 15 steps | Mid-term trend |
| `metric_0_mean_60` | 60 steps | Long-term baseline |
| `metric_0_std_5` | 5 steps | Short-term instability |
| `metric_0_std_15` | 15 steps | Mid-term instability |
| `metric_0_roc` | 1 step lag | Rate of change (spike detection) |

### Gold layer — model-ready features
- Selects 44 features: 38 raw metrics + 6 engineered rolling features
- Drops metadata columns (`source_file`, `time_step`, `ingested_at`)
- Writes separate Gold tables for train and test (test includes `is_anomaly` labels)

### Model training
- Algorithm: XGBoost classifier
- Class imbalance handled with `scale_pos_weight=23` (ratio of normal to anomaly rows)
- 70/30 train/eval split on labelled test data
- All runs tracked in MLflow: parameters, metrics, model artifact

### MLflow experiment tracking
- Logs: `n_estimators`, `max_depth`, `learning_rate`, `scale_pos_weight`
- Metrics: F1, AUC, Precision, Recall per run
- Best model registered in Databricks Model Registry as `smd-anomaly-detector`
- Champion alias assigned to the best-performing version

### Model serving
- Deployed via Databricks Model Serving as a real-time REST endpoint
- Accepts JSON payloads of feature vectors
- Returns `0` (normal) or `1` (anomaly) per row
- Validated end-to-end: 5/5 anomaly rows correctly predicted

### Drift monitoring
- Evidently AI generates weekly drift reports
- Compares incoming feature distributions against training baseline
- Metrics logged back to MLflow: `drift_share`, `dataset_drift`, `drifted_cols`
- Alert threshold: retrain triggered if >30% of columns show drift

---

## Tech stack

| Layer | Tools |
|---|---|
| Data platform | Databricks, Unity Catalog, Delta Lake |
| Data processing | PySpark, Spark SQL |
| Feature engineering | Rolling window functions, Databricks Feature Store |
| ML | XGBoost, scikit-learn |
| Experiment tracking | MLflow |
| Model registry | Databricks Model Registry |
| Model serving | Databricks Model Serving (REST endpoint) |
| Drift monitoring | Evidently AI |
| Orchestration | Databricks Workflows |
| Language | Python 3.12 |

---

## Results

| Metric | Value |
|---|---|
| AUC | 0.992 |
| F1 Score | 0.718 |
| Precision | 0.577 |
| Recall | 0.950 |
| Accuracy | 97% |
| Training rows | 495,894 |
| Evaluation rows | 212,526 |

**Why recall matters here:** In anomaly detection, missing a real failure (false negative) is more costly than a false alarm (false positive). The model is tuned to maximise recall — catching 95% of real anomalies at the cost of some false positives.

---

## Background

This project is directly motivated by real production experience. At Dubai Airports, I built data pipelines for infrastructure telemetry — ingesting CPU, memory, and network metrics from mission-critical systems. The data existed to predict failures before they happened, but the team was reacting to incidents rather than anticipating them.

This project adds the ML layer on top of that engineering foundation: taking the same type of telemetry data and building a fully automated system that detects anomalies in real time and retrains automatically when the data distribution shifts.

---

## Author

**Nadhiya Ganesan**
Data Engineer → MLOps | Databricks · Azure · PySpark

[LinkedIn](https://www.linkedin.com/in/nadhiya-ganesan93/) · [GitHub](https://github.com/Nadia0393)
