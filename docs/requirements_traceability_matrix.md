# Requirements Traceability Matrix (RTM)

### Project: OverSight ICS | Version: 1.2 | Status: Audit-Ready
**Linking Documents:** FAT-01 (HW), FAT-02 (FT/LG/DT), FAT-03 (ST/LG)

---

### 1. Hardware & Infrastructure (FAT-01)
| Req ID     | Requirement Description                                                                   | Validation Source |
|:-----------|:------------------------------------------------------------------------------------------|:------------------|
| **REQ-01** | **Network Isolation**: Network ports must be restricted to mission-critical traffic only. | **FAT-01: HW-01** |
| **REQ-02** | System must maintain sub-microsecond clock jitter for PTP sync.                           | **FAT-01: HW-02** |
| **REQ-03** | Ingest engine must handle high-velocity tag data without loss.                            | **FAT-01: HW-03** |
| **REQ-04** | Southbound link failures must be detected in under 2 seconds.                             | **FAT-01: HW-04** |
| **REQ-05** | Northbound connections must require valid TLS certificates.                               | **FAT-01: HW-05** |
| **REQ-06** | A hardware watchdog must trigger a Safe State on process hang.                            | **FAT-01: HW-06** |
| **REQ-07** | Hardware must reboot into a Safe State after a total power loss.                          | **FAT-01: HW-07** |
| **REQ-08** | Northbound (IT) traffic must be physically isolated from OT control.                      | **FAT-01: HW-08** |
| **REQ-09** | System must maintain PTP stability under 100% CPU thermal load.                           | **FAT-01: HW-09** |
| **REQ-10** | Storage media must meet industrial write-endurance standards.                             | **FAT-01: HW-10** |
| **REQ-11** | Chassis must be grounded to prevent EMI-induced signal noise.                             | **FAT-01: HW-11** |
| **REQ-12** | Redundant NICs must provide failover pathing (if applicable).                             | **FAT-01: HW-12** |
| **REQ-13** | LED indicators must provide physical status of the control loop.                          | **FAT-01: HW-13** |
| **REQ-14** | DIN-rail mounting must withstand specified vibration tolerances.                          | **FAT-01: HW-14** |

### 2. Functional & Logic (FAT-02)
| Req ID        | Requirement Description                                          | Validation Source |
|:--------------|:-----------------------------------------------------------------|:------------------|
| **REQ-LG-01** | AI must utilize $Z$-Score drift for anomaly detection.           | **FAT-02: FT-01** |
| **REQ-LG-02** | All AI troubleshooting must be cited from approved manuals.      | **FAT-02: FT-02** |
| **REQ-LG-03** | AI must state "No Information" for assets outside the RAG scope. | **FAT-02: FT-03** |
| **REQ-LG-04** | Every control intent must be cryptographically signed (Hash).    | **FAT-02: FT-04** |
| **REQ-LG-05** | Commands require physical feedback confirmation before clearing. | **FAT-02: FT-05** |
| **REQ-LG-06** | LangGraph must persist and recover state after a hard crash.     | **FAT-02: LG-01** |
| **REQ-LG-07** | Physical sensor values must override "Expected" software states. | **FAT-02: LG-02** |
| **REQ-LG-08** | Reasoning Agent must provide a logic audit for every "Stop".     | **FAT-02: LG-03** |
| **REQ-LG-09** | Graph nodes must execute in a deterministic, linear sequence.    | **FAT-02: LG-04** |
| **REQ-LG-10** | Database queries must return telemetry results in <500ms.        | **FAT-02: DT-01** |
| **REQ-LG-11** | AI must timeout and trigger an error if reasoning takes >30s.    | **FAT-02: DT-03** |

### 3. Safety & Security (FAT-03)
| Req ID     | Requirement Description                                           | Validation Source    |
|:-----------|:------------------------------------------------------------------|:---------------------|
| **REQ-01** | Semantic Gatekeeper must block writes to forbidden registers.     | **FAT-03: ST-01**    |
| **REQ-02** | System must prioritize critical alarms over HMI "Silent" modes.   | **FAT-03: ST-03**    |
| **REQ-03** | Sensitive strings (passwords) must be redacted from AI prompts.   | **FAT-03: ST-04**    |
| **REQ-04** | Loss of Agent-Gatekeeper heartbeat must trigger Orphan Mode.      | **FAT-03: ST-05**    |
| **REQ-05** | All orchestration requires Multi-Factor Dual-Authentication.      | **FAT-03: ST-06/07** |
| **REQ-06** | Unauthorized OPC-UA/CIP write attempts must be logged/blocked.    | **FAT-03: ST-08/09** |
| **REQ-07** | Control commands must be immune to Replay Attacks via Nonce.      | **FAT-03: ST-10**    |
| **REQ-08** | AI Agent must be resource-jailed to 40% CPU max.                  | **FAT-03: ST-11**    |
| **REQ-09** | Semantic layer must neutralize prompt-injection attempts.         | **FAT-03: ST-12**    |
| **REQ-10** | VFD and Pump assets must respect hard-coded range limits.         | **FAT-03: LG-05**    |
| **REQ-11** | System must enforce a 60s motor "Cool Down" interlock.            | **FAT-03: LG-06**    |
| **REQ-12** | PTP Signal loss must force the Southbound Driver into Safe State. | **FAT-03: LG-07**    |