# Unified Namespace (UNS) Data Schema

## 1. Namespace Hierarchy
OverSight ICS uses a standardized topic structure to decouple high-level AI logic from low-level hardware addresses. 

**Structure:** `Enterprise/Site/Area/Line/Cell/Tag_Category/Asset_Name`

### 1.1 Topic Categories

- **Telemetry**: Real-time process variables (e.g., Pressure, Flow, Temperature).
- **Commands**: AI-generated control intents (e.g., Start, Stop, Setpoint Change). 
- **Status**: Current operational status (e.g., Running, Faulted, Local/Remote).
- **Metadata**: Static Asset Information, audit dates, and ownership records.

**Example:** `AcmeCorp/Longview/WaterTreatment /Filtration/Pump_01/Telemetry/Discharge_Pressure`

## 2. Message Payload Specification
All data exchange occurs via JSON payloads wrapped in Sparkplug B envelopes to maintain state awareness (Birth/Death Certificates).

### 2.1 Telemetry Payload (Read-Only)
Sent from Level 2 to the UNS.

```json
{
  "timestamp": "ISO-8601",
  "asset_id": "PUMP_01",
  "metrics": [
    { 
      "name": "flow_rate", 
      "value": 450.5, 
      "unit": "GPM", 
      "quality": 192,
      "eng_range": { "min": 0, "max": 1000 }
    },
    { 
      "name": "cabinet_tamper", 
      "value": 0, 
      "unit": "bool",
      "quality": 192 
    }
  ],
  "metadata": { 
    "asset_status": "running", 
    "mode": "auto",
    "location": "Sector_7G_Cabinet_A12"
  }
}
```
### 2.2 Asset Metadata Payload (Low-Frequency)
Published to the `.../Metadata/Asset_Name` branch.

```json
{
  "asset_id": "PUMP_01",
  "owner": "Lead_OT_Engineer",
  "classification": "Critical_Infrastructure",
  "installation_date": "2025-11-20",
  "last_audit_date": "2026-01-15",
  "next_audit_due": "2027-01-15",
  "service_manual_ref": "VFD-AB-04-A"
}
```

### 1.3 Timestamp Precision Standards
- **L3/L4 Payloads**: Use ISO-8601 with millisecond precision (e.g., `2026-04-16T14:00:00.123Z`).
- **L2 Telemetry (PTP-backed)**: Use Unix Nanoseconds for internal $Z$-Score drift calculations to ensure jitter 
  doesn't trigger false anomalies.

## Revision History
| Version | Date       | Author    | Description of Change                                           | Approved By |
|:--------|:-----------|:----------|:----------------------------------------------------------------|:------------|
| 1.0     | 2026-04-12 | armummert | Initial Baseline for Version 1.0 Release.                       | armummert   |