# SOP-SAFE-ENVELOPE-01: Defining Safe Operating Envelopes

### 1. Objective
This SOP provides the template for defining the **Safe Operating Envelope (SOE)** for each asset type, ensuring OverSight ICS guardrails are grounded in physical mechanical limits [cite: Auditor's Findings].

### 2. Envelope Definition Template
| Boundary Category | Parameter | Physical Limit (Source) | OverSight Guardrail |
| :--- | :--- | :--- | :--- |
| **Critical High** | Max Operating Speed | e.g., 3600 RPM (OEM Manual) | **Forbidden Register Write** > 3500 RPM [cite: CHANGE_MANAGEMENT_LOG.md]. |
| **Dynamic Drift** | Discharge Pressure | e.g., 150 PSI (Design Spec) | **Z-Score Alert** at +2.0σ [cite: 3.1]. |
| **Safety Interlock** | Bearing Temp | e.g., 180°F (Critical Fail) | **Pessimistic Confirm** required at 165°F [cite: 3.1]. |

### 3. Recording Procedure
1.  **Mechanical Review**: Extract "Never Exceed" limits from OEM technical manuals [cite: Auditor's Findings].
2.  **Baseline Generation**: Run asset in "Nominal" mode for 24 hours to establish mean/standard deviation [cite: 3.1].
3.  **Boundary Mapping**: Configure the **Semantic Gatekeeper** to block writes exceeding OEM limits [cite: COMMISSIONING.md].
4.  **Verification**: Execute **OAT-01 (Gatekeeper Block)** to confirm enforcement [cite: VALIDATION_PROTOCOLS.md].
