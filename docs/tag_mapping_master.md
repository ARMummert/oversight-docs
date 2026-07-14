# Oversight ICS: TAG MAPPING MASTER



This document defines the standardized data structure for mapping physical field asset registers to the Unified 
Namespace (UNS).  This mapping is critical for the AI Reasoning Agent, so it can interpret the telemetry and issue 
intents across hardware environments.

## 1. Unified Namespace (UNS) Hierarchy
All tags must be published to the MQTT broker (Sparkplug B) following the ISA-95 hierarchical structure. 

**Topic Structure:** `Enterprise/Site/Area/Line/Cell/Tag_Category/Asset_Name`

**Example:** `AcmeCorp/Longview/WaterTreatment/Filtration/Pump_01/Telemetry/Discharge_Pressure`

## 2. Tag Classification Matrix
Tags are grouped into functional categories to enforce security and operational boundaries.

| Category     | Type       | Description                                                                |
|:-------------|:-----------|:---------------------------------------------------------------------------|
| Telemetry    | Read-Only  | Real-Time process variables (Pressure, Flow, Temp)                         |
| Commands     | Write-Only | AI-generated intents (Start, Stop, Setpoint, Change                        |
| Status       | Read-Only  | Asset state (Running, Faulted, Local/Remote)                               |
| Safety (SIS) | FORBIDDEN  | Hard-coded safety interlocks. The AI Agent is prohibited from writing here |
| Diagnostics  | Read-Only  | System Health (CPU Temp, Network Latency, Heartbeat)                       | 


## 3. Mandatory Metadata Schema (JSON)

```json
{
  "timestamp": "ISO-8601",
  "asset_id": "PUMP_01",
  "metrics": [
    {
      "name": "discharge_pressure",
      "value": 48.5,
      "unit": "PSI",
      "quality": 192,
      "eng_range": { "min": 0, "max": 100 }
    }
  ],
  "metadata": {
    "mode": "auto",
    "loto_status": false
  }
}
```

## 4. Hardware Mapping Rules
To maintain system integrity during commissioning, the following mapping rules are enforced:

### 4.1 Quality Bit Validation

- **Telemetry (Read)**: The Anomaly Engine ingests only tags with a quality bit $\ge 192$ (Good).
- **Commands (Write)**: The Southbound Driver must not consider a command "SUCCESS" based solely on the write operation.
- **Validation Logic**: Every `WRITE_REGISTER` intent must be paired with a corresponding `WAIT_FOR_VALUE` or 
  `STATUS_CHANGE`. If the feedback quality bit is $< 192$ or the value does not update within the `timeout_ms` defined 
  in the intent payload, the intent is marked as FAILED in the `intent_outbox`.

### 4.2 Physical Isolation (Dual-NIC)

- **Southbound (NIC 2)**: All raw PLC registers are mapped exclusively to the Southbound Driver.
- **Northbound (NIC 1)**: Only sanitized UNS topics are available to the Reasoning Agent and HMI.

### 4.3 Maintenance & LOTO Logic

- **Hardware Interlock**: Each asset mapping must include a Local_Hand_Switch bit.
- **Effect**: If the physical switch at the cabinet is in "Hand" or "Off," the Southbound Driver will return a `403 
  FORBIDDEN_HARDWARE_LOTO` error for any incoming AI intents.

## 5. Master Asset Registry (Example)

| Asset   | Tag Name  | UNS Path                                                                  | DataType |
|:--------|:----------|:--------------------------------------------------------------------------|:---------|
| PUMP_01 | CMD_START | `AcmeCorp/Longview/WaterTreatment/Filtration/Pump_01/Commands/CMD_START`  | BOOLEAN  |
| PUMP_01 | Flow_Rate | `AcmeCorp/Longview/WaterTreatment/Filtration/Pump_01/Telemetry/Flow_Rate` | REAL     |
| PUMP_01 | Position  | `AcmeCorp/Longview/WaterTreatment/Filtration/Pump_01/Telemetry/Position`  | INT      |
| PUMP_01 | SIS_Trip  | `AcmeCorp/Longview/WaterTreatment/Filtration/Pump_01/Safety/SIS_Trip`     | BOOLEAN  |

## Revision History
| Version | Date       | Author    | Description of Changes                    | Approved By |
|:--------|:-----------|:----------|:------------------------------------------|:------------|
| 1.0     | 2024-06-01 | armummert | Initial Baseline for Version 1.0 Release. | armummert   |