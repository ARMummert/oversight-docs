# OverSight ICS Evaluation Framework

## 1. Purpose

The purpose of this document is to define the formal evaluation framework used to measure the performance, 
reliability, and accuracy of the Oversight ICS Cognitive Reliability Agent. By calculating the **F1 Score** of the 
Oversight Agent, we can establish the **Ground Truth** for industrial intent generation and the criteria for binary 
classification.

All framework performance claims (e.g., F1 Score ≥ 0.92) are **reproducible**, **auditable**, and **traceable**.

## 2. Scope

This framework applies to AI-generated alarms, predictive failover intents, and anomaly detection outputs. 
It does **NOT** apply to Safety Instrumented Systems (SIS) or Hardwired Interlocks.

## 3. Prediction Objectives

The OverSight system generates predictions in three categories:

| Category              | Description                                  | Output Type      |
|-----------------------|----------------------------------------------|------------------|
| Failure Prediction    | Impending asset failure or fault condition   | Failover Intent  |
| Degradation Detection | Performance decline requiring intervention   | Advisory / Alarm |
| Anomaly Detection     | Statistical deviation from baseline behavior | Alarm            |

Each prediction includes a`timestamp`, `asset_id`, `prediction_type`, `confidence_score` (SCC), and`trace_id`
(for auditability).

## 4. Ground Truth Definition

Ground truth events are defined using **Multi-Source Validation**:

### 4.1 Sources of Ground Truth

1. **Sources**: CMMS(Central Management System) records, Historian data, FAT/SAT test events, and commissioning phase
   observations
2. **Synthetic Telemetry**: Modeled after Brownian Noise (BN) or specific sensor jitter.
3. **Adversarial Testing**: RAG test documents containing poisoned instructions to test the AI-04 Alignment Gate.
4. **Human-In-The-Loop**: Historial `trace_id` logs with positive operator attribution.

### 4.2 Event Structure

Each ground truth event must include:

- `event_id` and `asset_id`
- `event_type` (failure, degradation, anomaly)
- `start_time` and `end_time`
- `validation_source`
- `confidence_level` (operator / automated / test)

## 5. Classification & Temporal Mapping

### 5.1 Classification

Predictions are evaluated as **True Positive (TP)**, **False Positive (FP)**, **False Negative** or **True Negative (TN)**.

| Classification      | Definition                                                                      | Meaning in Oversight ICS                                                                         | System Impact                      |
|---------------------|---------------------------------------------------------------------------------|:-------------------------------------------------------------------------------------------------|:-----------------------------------|
| True Positive (TP)  | Prediction occurs AND matching ground truth event exists within Δt (delta time) | Agent Correctly identifies a $Z$-Score drift and suggests the pre-verified manual failover plan. | Process Stabilized                 |
| False Positive (FP) | Prediction occurs BUT no matching ground truth event                            | Agent suggests a failover plan for a nominal process ($Z < 1.5$) or "Hallucinates" a fault.      | Nuisance Alarm / Unnecessary Wear  |
| False Negative (FN) | Ground truth event occurs BUT no prediction                                     | Agent fails to draft an intent or cite RAG context during a valid $Z$ > 3.0 event.               | **CRITICAL FAILURE / SAFETY TRIP** |
| True Negative (TN)  | No prediction AND no event                                                      | Agent remains in **background monitoring** while the process is stable.                          | Baseline Operation                 |

### 5.2. Temporal Matching Window (Δt)

A prediction is considered **correct** if:

```
prediction_time ∈ [event_start - Δt, event_start + Δt]
```

Where:

- **Default Δt** = ±5 minutes
- **Detection Latency Target**: $\le 120$ seconds for critical assets.

### 6. Formal Requirements & Schema Enforcement
To maintain alignment with the Oversight ICS [Alarm Philosophy](/docs/alarm_philosophy.md) the following requirements
are strictly enforced:

### 6.1 Global Reliability Constant
- **Statistical Correlation Score (SCC)**: Must be $\ge 85\%$ for an intent to be classified as high confidence. 
  Verified as $1 - p\text{-value}$ of the $Z$-drift. 
- **$Z$-Score Alarm Limit: $Z \le 3.0$**
- **$Z$-Score Hysteresis: $Z \le 2.0$**
- **Detection Latency Max**: $> 120$ seconds for critical assets.

### 6.2 Alarm Management & Suppression
- **Max Shelf Life**: Configuration files **MUST** include `MAX_SELF_DURATION_MINUTES: 240`.
- **Max Concurrency Shelves**: The system limits the number of concurrent shelved alarms to 5. If the limit is reached,
 a `SHELVED_ALARM_CAPACITY_EXCEEDED` advisory is issued.
- **Alarm Flooding**: Events with $> 10$ alarms in $10$ minutes are excluded from standard scoring and flagged for a 
  mandatory Alarm Rationalization Review.

### 6.3 Operational Locks (FMEA Alignment)

- **Stability Lock**: A 15-minute (900s) lock on new alarms for a specific tag following an autonomous threshold update
  to prevent chattering.
- **I-term Reset Lock**: A 120-second period for integral-term stabilization following a state change.
- **Pessimistic Timeout**: A 5-second network handshake timeout for Southbound Driver confirmation.

### 6.4 KPI & Distribution Monitoring

- **Critical Distribution Cap**: If **Critical** alarms exceed 10% of the total distribution over any 4-hour period, the
  system issues a `KPI_DISTRIBUTION_DEVIATION` advisory.
- **Alarm Flood Threshold**: Triggers if >10 alarms occur within 10 minutes.
- **Stale Alarm Threshold**: Flagged for intervention if active beyound 24 hours.

## Data Set Requirements

```json
{
  "dataset_version": "2026.04.15",
  "plant_id": "PLANT-001",
  "assets_included": 42,
  "time_range": "30 days",
  "label_sources": ["CMMS", "Historian", "Operator"]
}
```

## 8. Metrics

### 8.1 Core Metrics

Precision:

```
Precision = TP / (TP + FP)
```

Recall:

```
Recall = TP / (TP + FN)
```

F1 Score:

```
F1 = 2 × (Precision × Recall) / (Precision + Recall)
```

### 8.2 Additional Metrics

| Metric              | Description                   | Target |
|---------------------|-------------------------------|--------|
| False Positive Rate | FP / (FP + TN)                | ≤ 5%   |
| Detection Latency   | Time to detect event          | ≤ 120s |
| Alarm Accuracy      | Correct alarms / total alarms | ≥ 90%  |

## 9. Acceptance Criteria

The system is considered **production-ready** when:

* F1 Score ≥ **0.92**
* False Positive Rate ≤ **5%**
* Detection Latency ≤ **120 seconds**
* No critical missed failures (FN) in validation dataset

## 10. Evaluation Procedure

### 10.1 Standard Evaluation Flow

1. Load versioned dataset
2. Run the system in **Shadow Mode** (no control authority)
3. Capture predictions
4. Match predictions to ground truth events
5. Classify TP / FP / FN / TN
6. Compute metrics
7. Store results

### 10.2 Shadow Mode Requirements

1. No commands are issued to control systems
2. Full reasoning pipeline active
3. All predictions logged

### 10.3 Output Storage

Results must be stored in:

* Audit log system
* Evaluation report database
* Versioned evaluation artifacts

## 11. Audit & Traceability

All evaluation runs must include:

* `dataset_version`
* `model_version`
* `knowledge_version`
* `timestamp`
* `operator_id` (if manual)

### 11.1 Traceability Requirement

Each prediction must be traceable via:

* `trace_id`
* Associated reasoning logs
* Source telemetry data

## 12. Evaluation Frequency

| Phase                | Frequency               |
|----------------------|-------------------------|
| Commissioning        | Initial validation run  |
| Production           | Monthly                 |
| After Model Update   | Mandatory re-evaluation |
| After Dataset Update | Mandatory re-evaluation |

## 13. Model Drift Monitoring

### 13.1 Drift Indicators

* F1 Score drop > 5%
* Increase in False Positives
* Change in baseline Z-score distribution

### 13.2 Drift Response

* Trigger investigation
* Re-run evaluation
* Rollback model if required
* Notify engineering team

## 14. Limitations

* Performance depends on data quality
* Sensor failures may impact accuracy
* Rare events may be underrepresented
* Ground truth labeling may include human bias

## 15. Related Documents

* README.md
* SOP-AI-01 RAG Ingestion
* FAT-03 Test Protocol
* TELEMETRY_DATABASE_SCHEMA.md
* AUDIT_LOGGING.md
* CHANGE_MANAGEMENT_LOG.md

## 16. Final Statement

> All performance claims made by OverSight ICS must be derived from this framework.

Unverified or non-reproducible metrics are **not permitted** in any customer-facing material.
