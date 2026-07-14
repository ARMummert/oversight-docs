# Operational Security Procedures

## Revision History
| Version | Date       | Author       | Description of Change                                           | Approved By |
|:--------|:-----------|:-------------|:----------------------------------------------------------------|:------------|
| 1.0     | 2026-04-12 | [User]       | Initial Baseline for Version 1.0 Release.                       | [Manager]   |

## 1. Access Review Schedule (ISO 27001 A.9)
To prevent "Permission Creep," all service accounts and human access tokens are subject to periodic validation.
- **Cycle**: Every 90 days (Quarterly).
- **Procedure**: 
    1. Generate a report of all active JWT Service Accounts from the OIDC provider.
    2. Review the `trace_id` history in the **Audit Logs** to verify the account has performed work in the last 30 days.
    3. Revoke any accounts associated with decommissioned agents or inactive personnel.
- **Automation**: The `compliance_watcher.py` script triggers a `REVIEW_REQUIRED` flag in the SIEM if an account 
  remains active without an associated audit entry for >45 days.

## 2. Information Asset Register (ISO 27001 A.8)
This register maps technical assets to business owners and data classifications defined in the CSMP.

| Asset ID              | Physical Location | Data Owner       | Classification | Impact of Loss       |
|:----------------------|:------------------|:-----------------|:---------------|:---------------------|
| **Edge Gateway 01**   | Main Control Room | OT Manager       | Restricted     | High (System Down)   |
| **Southbound Driver** | Cabinet A-12      | Lead Engineer    | Critical       | Very High (Safety)   |
| **RAG Vector DB**     | IDMZ Volume 1     | Data Architect   | Internal       | Medium (Logic Drift) |
| **Audit Log Volume**  | WORM Storage      | Security Officer | Critical       | High (Compliance)    |

## 3. Physical Security & Tamper Monitoring
Physical access to the Edge Gateway and Control Cabinets is monitored via telemetry.
- **Cabinet Tamper Switch**: All OverSight-enabled cabinets must be equipped with a physical limit switch connected to 
  the PLC.
- **Telemetry Tag**: `Enterprise/Site/Area/Line/Cell/Safety/Cabinet_Door_Open`.
- **Logic**: If the cabinet door is opened without a scheduled "Maintenance Mode" bit active, the AI Agent must log a 
  `PHYSICAL_SEC

- ## 4. AI Risk Assessment & Impact Policy (ISO 42001 / NIST AI RMF)
This section defines the risk profile for the Agentic Reasoning layer and the controls used to mitigate physical-world 
consequences.

### 4.1 Risk Categorization
| Risk Type            | Industrial Impact                                                        | Mitigation Control                                                                                                          |
|:---------------------|:-------------------------------------------------------------------------|:----------------------------------------------------------------------------------------------------------------------------|
| **Hallucination**    | AI proposes a setpoint that exceeds physical equipment design.           | **$Z$-Score Grounding**: The Semantic Gatekeeper rejects any value >3 standard deviations from the baseline.                |
| **Prompt Injection** | Malicious actor manipulates RAG sources to trigger a shutdown.           | **Signed RAG Sources**: Only PDF manuals with a verified SHA-256 hash are ingested into the Vector DB.                      |
| **Logic Drift**      | AI fails to recognize a sensor failure, treating it as a process change. | **Sensor Fusion**: The agent must correlate at least two telemetry points (e.g., Amps + Flow) before initiating a failover. |
| **Latency/Jitter**   | AI reasoning takes too long, missing a critical failover window.         | **Cgroup Isolation**: Python process is locked to 40% CPU to ensure the Watchdog Heartbeat is never starved.                |

### 4.2 Impact Assessment (The "Pessimistic Confirmation" Rule)
Any AI intent that results in a **Level 2 (Physical Change)** must follow the impact-based authorization workflow:

1. **Low Impact (Advisory)**: AI suggests a maintenance task. → *Auto-log in Maintenance Management System.*
2. **Medium Impact (Parameter Change)**: AI suggests adjusting a setpoint by <10%. → *Requires Operator "Confirm" on 
   HMI.*
3. **High Impact (Asset Failover)**: AI suggests stopping a Primary Pump and starting a Standby. → 
   *Requires "Two-Key" verification: AI Reasoning Trace + Human Operator Credentials + Semantic Gatekeeper Validation.*

### 4.3 Bias & Fairness Monitoring
The AI Agent must monitor the "Bias" of the system. 
In an ICS context, "Bias" refers to the model favoring certain operating modes (e.g., favoring Pump A over Pump B 
regardless of wear). 
- **Control**: The system logs "Selection Frequency." If the AI selects one asset 20% more than its twin without a 
  $Z$-score justification, a **Logic Drift Alarm** is triggered for engineering review.

## 5. Change Management & Continuity (ISO 27001)
### 5.1 Software Change Control
All changes to the **Southbound Driver** or **Semantic Gatekeeper** must follow the **Three-Tier Deployment** process:
1. **Virtual Twin**: Verify $Z$-score logic against simulated process noise.
2. **Staging**: Deploy to a non-critical asset with a 24-hour **Monitor Only** window.
3. **Production**: Deploy during a scheduled maintenance window with Lead OT Engineer approval.

### 5.2 Disaster Recovery (DR)
- **Recovery Time Objective (RTO)**: 4 Hours.
- **Procedure**: In the event of Edge Gateway hardware failure, the system falls back to **Local/Manual** control. A 
  pre-imaged cold-standby IPC is kept on-site in Cabinet A-12 for immediate swap.

