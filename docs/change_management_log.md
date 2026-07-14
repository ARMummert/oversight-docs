# Change Management Log

### 1. Objective

This log serves as the official audit for all modifications to the Oversight ICS production environment. Every change
(software patch, a RAG context update, security update, or a modification to the **Semantic Gateway**) is documented, 
authorized, and reversible. This is required to maintain safety-dominance of the system and prevent unauthorized logic 
drift.

### 2. Governance & Approval Matrix

| Change Risk | Threshold / Type of Modification                          | Required Approval               | Required Verification                                                                                                |
|-------------|-----------------------------------------------------------|---------------------------------|----------------------------------------------------------------------------------------------------------------------|
| Low         | $\Delta \le 5\%$; Documentation; Metadata; UI Changes     | System Auto-Authorized          | `trace_id` is logged in the audit logs + HMI Advisory notification.                                                  |
| Medium      | $\Delta$ 5% – 15%; RAG Ingest (New Manuals); DB Policies. | Lead Engineer                   | Manual "commit" on HMI; [SAT-04: Post-Production Integration](/docs/sop/SOP-SYS-002_Post-Production-Integration.md).      |
| High        | $\Delta > 15\%$; Forbidden Register List; Core Driver.    | Two-Key (Engineer + Manager)    | [SAT-05 Security & Safety Regression](/docs/sop/SOP-SEC-001_Safety-Regression-Testing.md)                            |
| High        | **Forbidden Register List**; Edit                         | Safety Lead                     | FMEA Revision Review; Witness of [FAT-03](/docs/fat/FAT-03-safety_and_security_protocol.md)                          |
| Medium      | **RAG Ingest** (New Tech Manuals)                         | Lead Engineer                   | Semantic check of citations in HMI.                                                                                  |
| High        | **Hardware/Power Configuration**                          | Two-Key (Safety Lead + Manager) | Redundancy Failover Test; Witness of [Change Regression Test Protocol](/docs/change_regression_test_protocol.md) |                     

### 3. Change Request (CR) Protocol

1. **Submission**: Submit CR with a description and a verified Docker image hash or database snapshot.
2. **Assessment**: Evaluate impact on deterministic timing, PTP synchronization, and FMEA Alignment.
3. **Approval**: Authorization obtained based on the Matrix in Section 2.
4. **Implementation**: Change applied to the Edge Gateway.
5. **Validation (PIR)**: Post-implementation Review (PIR) to guarantee no unintended $Z$-Score drift or Heartbeat loss.
6. **Rollback (SOP-03: Rollback Procedure)**: If PIR fails, the system is reverted via 
   [SOP-03: Rollback Procedure](/docs/sop/SOP-SYS-001_Backup-Rollback.md) using the pre-verified Docker image hash or 
   database snapshot.

### 4. Change Request (CR) Protocol Pre-Requisite Checklist
Before any entry is made to the log below, the following criteria must be met:

- [ ] **Risk Assessment**: Changes to the **Forbidden Register List** are classified as **High Risk** and require a
      500ms **Safe State** validation.
- [ ] **FMEA Update**: High-risk changes to asset limits (e.g., VFD speed-lock) must be documented in the FMEA
      (Failure Mode and Effects Analysis Log).
- [ ] **SAT Mapping**: Identify the specific Site Acceptance Test (SAT) required for verification.
- [ ] **Rollback Plan**: Verify Docker image hash or database snapshot is available in the local respository.
- [ ] **Impact Analysis**: Evaluate if the change affects the **Southbound Driver's** deterministic timing or PTP
      synchronization.

### 5. Example Change Management Log Table
| Date       | CR ID  | Description of Change                                                                | Risk Level | Verification Status and CRT | PIR Complete | PIR Signoff Date |
|------------|--------|--------------------------------------------------------------------------------------|------------|:----------------------------|:-------------|:-----------------|
| 2026-04-14 | CR-001 | Patching FastAPI to v0.109.1 (Security)                                              | Low        | **PASS** (CRT-03)           | YES          | 2026-04-14       | 
| 2026-04-16 | CR-002 | Updated `forbidden_list` for new VFD speed-lock                                      | High       | **PASS** (CRT-01)           | YES          | 2026-04-15       |
| 2026-04-20 | CR-003 | Ingested `PUMP_01` manual into RAG vector space                                      | Medium     | **PASS** (CRT-02)           | YES          | 2026-04-17       |
| 2026-04-22 | CR-004 | Adjusted TimescaleDB retention to 120 days                                           | Low        | **PASS** (CRT-09)           | YES          | 2026-04-20       |
| 2026-04-23 | CR-005 | Threshold Adaptation: Adjusted `PUMP_01_DISCHARGE` $Z$-Score from 1.5 to 1.65 (+10%) | Medium     | **PASS** (CRT-02)           | YES          | 2026-04-21       |

### 6. Emergency Change Procedure
In the event of a P1 Incident (Security Breach or Logic Failure) as defined in the Incident Response Plan, an emergency 
rollback to the "Last Known Good" state can be performed via 
[SOP-03: Rollback Procedure](/docs/sop/SOP-SYS-001_Backup-Rollback.md). An Emergency CR must be filed within 4 hours of 
the action to document the recovery and maintain the audit trail.
