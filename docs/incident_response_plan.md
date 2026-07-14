# Incident Response Plan: Cybersecurity & Operational Response

### 1. Objective
This plan defines the immediate actions and recovery procedures for the Oversight ICS environment when a security
breach, AI Logic Failure, or "Out-of-Bounds" process event is detected. The goal of this plan is to maintain the 
**Safe State** of field assets and ensure forensic integrity.


### 2. Emergency Contacts

| Role                           |        Contact Name        | Phone Number   |
|:-------------------------------|:--------------------------:|:---------------|
| **Site Safety Lead**           |     On-Call Shift Lead     | 360-555-2345   |
| **ICS Security Engineer**      | Lead CyberSecurity Officer | 360-343-3456   |
| **Local Emergency (Fire/EMS)** |        911 Dispatch        | 360-685-9304   |
| **Oversight Vender Support**   |       Oversight 24/7       | 1-800-OT-GUARD |
| **Regulatory (CISA)**          |        Central Desk        | 1-888-282-0870 |

### 2. Incident Classification Priority Levels
The Oversight ICS environment is classified as follows:

1. **Critical**: The Semantic Gatekeeper is unable to process any intents.
2. **High**: The Southbound Driver detects a loss of heartbeat from the field asset or an unauthorized modification to the `knowledge_vector` or `asset_health` views.
3. **Medium**: Anomalies in the RAG context or evidence of logic drift without immediate safety implications.
4. **Low**: Only for documentation, UI updates, and non-security-related log events.

| Level | Classification | Trigger Example                                                                 | Response Action                                                                                                      | 
|:------|:---------------|:--------------------------------------------------------------------------------|:---------------------------------------------------------------------------------------------------------------------|
| P1    | Critical       | Semantic Gatekeeper block, Heartbeat loss, or Unauthorized Physical NIC bridge. | **IMMEDIATE E-STOP**: AI lockout activated. Manual Override Required. Notify CISA/Regulatory bodies within 24 hours. |
| P2    | High           | $Z$-Score drift > 3.0 without RAG justification, Malware detected on Gateway.   | **ISOLATION**: Disconnect Northbound (IT) network. Perform memory dump of Reasoning Agent for forensic analysis.     |
| P3    | Medium         | Unauthorized modification to asset_config or forbidden_registers tables.        | **FORENSIC PRESERVATION**: Freeze SQL WAL logs; trigger SIEM Alert SEC-RAG-01; notify Cybersecurity Lead.            |
| P4    | Low            | Minor documentation typos, non-security log rotation, or UI aesthetic updates.  | **MONITORING**: Log the event for trend analysis. No immediate action required unless frequency exceeds threshold.   |

### 2.1 Incident Response Procedures (P1 Critical)

### 3. Immediate Response Flow (Supply Chain / RAG Attack)
In the event of a Critical (P1) or High (P2) incident, the following protocol is enforced by the Southbound Driver:

- **1. AI Lockout**: The Southbound Driver stops polling the `intent_outbox`.
- **2. Deterministic Fallback**: The field asset heartbeat bit is dropped. The physical asset logic enters a 
"Safe State" within 500ms.
- **3. HMI Notification**: The operator dashboard transitions to a Red Strobe state with error code: 
`SECURITY_VIOLATION_LOCKOUT`.

If unauthorized modification to the `knowledge_vector` or `asset_health` view is detected:

- **1. Immediate Lock**: The `rag_ingest_pipeline` service is suspended.
- **2. SIEM Integration**: An automated alert is sent to the corporate SOC (Security Operations Center) via the 
  **Syslog/SIEM** conduit.
- **3. Forensic Snapshot**: A read-only snapshot of the PostgreSQL instance is created. Do **NOT** re-verify hashes until 
  the malicious delta is identified and logged for the **Post-Mortens** analysis.
- **4. Validation**: Compare the current vector table against the last signed SPDX SBOM and the WORM-storage backup.

### 4. Containment & Eradication

### 4.1 Digital Containment

- **Network Isolation**: Physically disconnect the Northbound NIC 1.
- **Process Isolation**: Switch all field assets to "Local/Hand" mode to bypass the Edge Gateway entirely.

### 4.2 Eradication of Malicious Logic

- **Snapshot Analysis**: Compare the current service hashes against the verified software bill of materials.
- **Database Rollback**: Revert telemetry and `intent_outbox` to the last known good state using TimescaleDB snapshots.

### 5. Forensic Investigation & Recovery

### 5.1 Reasoning Trace Audit

- Review the `trace_id` associated with the incident in the `intent_outbox` to determine if the failure was adversarial
  (external spoofing) or algorithmic (logic drift).

### 5.2 Recovery Validation
Before returning to **AUTO/AI** mode:

1. Verify the Semantic Gatekeeper configuration and the Forbidden Register List have not been tampered with.
2. Run a 1-hour **Shadow Mode** test where the AI suggests intents but the Southbound Driver does not execute them.

### 5.3 Escalation & Regulatory Notification 
In accordance with federal and industrial guidelines (e.g., CISA CIRCIA), the following notifications are mandatory
for P1 and P2 events:

- **CISA Central**: Report via the official portal if the incident results in a loss of control or potential safety 
impact to critical infrastructure.
- **EPA / Environmental**: Notify if the incident potentially impacts effluent discharge quality (for Water Reclamation 
Plants).
