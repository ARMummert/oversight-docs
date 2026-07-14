# SOP-01: Recurring Maintenance Procedures

### 1. Objective
This Standard Operating Procedure (SOP) defines the recurring maintenance tasks that are required for the long-term
reliability of the Oversight ICS data layer, the accuracy of the Reasoning Agent, and the physical integrity of the
Edge Gateway hardware.

### 2. Database & Data Layer Management
The following tasks are critical for TimescaleDB and pgVector to maintain performance targets defined in the
[Telemetry Database Schema](/docs/telemetry_database_schema.md).

| Frequency | Task ID | Description                                                                                                                                                                                                                                                                               | Targeted Outcome                                                              |
|-----------|---------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------|
| Weekly    | DB-01   | Verify TimescaleDB "Chunk" compression is active on the `telemetry` hypertable.                                                                                                                                                                                                           | Storage Optimization and disk pressure relief.                                |
| Monthly   | DB-02   | Execute a `VACUUM ANALYZE` on the `knowledge_vector` table.                                                                                                                                                                                                                               | Optimized query planning for RAG similarity searches.                         |
| Quarterly | DB-03   | **Z-Score Baseline Audit**: Manually trigger the `recalculate_baselines()` script using a 30-day window of "Steady State" telemetry. **Validation**: The new $\mu$ and $\sigma$ must be compared against the previous quarter; shifts $>15\%$ require a signed Engineering Justification. | Prevents "Alarm Fatigue" or "Missed Anomalies" due to seasonal process drift. |

### 3. AI & RAG Context Maintenance
To prevent **Logic Drift**, the Reasoning Agent's context must be synchronized with the actual physical state of the 
plant.

| Frequency     | Task ID | Description                                                                                                                                   | Targeted Outcome                                                                |
|---------------|---------|-----------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------|
| Monthly       | AI-01   | Re-index the RAG Vector database with any new technical bulletins from VFD/PLC manufacturers.                                                 | Prevention of hallucinations caused by outdated technical manuals.              |
| Monthly       | AI-02   | Review a random sample of `trace_id` logs from the `intent_outbox` for justification accuracy.                                                | Continuous improvement of the Reasoning Agent's logic.                          |
| Quarterly     | AI-03   | Re-calculate $Z$-Score baselines for all assets to account for seasonal process changes.                                                      | Reduction in "Alarm Fatigue" caused by shifting baseline noise.                 |
| Semi-Annually | AI-04   | **Forbidden Register List Audit**: Compare the current `forbidden_list.json` against the site's most recent P&ID and Electrical Safety Study. | Ensures safety-critical interlocks are not bypassed by new equipment additions. |

### 4. Hardware & Security Maintenance
To keep the Industrial PC (IPC) a hardened deterministic host for the Southbound Driver.

| Frequency | Task ID | Description                                                                                                                                                                                                     | Targeted Outcome                                                   |
|-----------|---------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------|
| Weekly    | HW-01   | Inspect the physical dual-NIC cables for wear or unauthorized bridge attempts.                                                                                                                                  | Maintenance of physical IT/OT network isolation.                   |
| Monthly   | HW-02   | Compare running service hashes against the [SBOM Compliance Report](/docs/sbom_compliance_report.md)                                                                                                            | Detection of unauthorized firmware or software modifications.      |
| 180 Days  | HW-03   | **TPM Key Rotation & Watchdog Test**: Rotate the TPM 2.0 Root of Trust key per CSMP Section 11. Manually trip the Hardware Watchdog Relay to verify physical de-energization of the Master Control Relay (MCR). | Cryptographic security and physical fail-safe verification.        |  
| Annual    | HW-04   | **Cold Start Recovery Test**: Perform a full system restore from the last stable recovery image. Pass Criteria: Full operational readiness (Reasoning Agent + Southbound Driver active) in <15 minutes.         | Validation of **Disaster Recovery** and **State of Grace** timing. |
