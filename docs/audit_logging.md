# Comprehensive Agent Interaction Logging

**Security Level**: `Level 3.5 IDMZ`
**Framework Alignment**: NIST 800–82 AU / ISA-62443 / NERC CIP-007-6 / EPA 40 CFR Part 141 / SIP
**Version**: 1.2

To satisfy legal and regulatory requirements, Oversight ICS maintains an immutable, multi-layered 
audit trail of every interaction between the AI, the human operator, and the physical hardware.

## 1. Regulatory Retention Requirements
To meet legal and regulatory requirements compliance across diverse industrial sectors, Oversight ICS enforces a
**3-year minimum retention policy**.  This is mandated by the following frameworks:

| Regulation / Framework  | Industry Focus         | Requirement                                                                 |
|:------------------------|:-----------------------|:----------------------------------------------------------------------------|
| **NIST 800-82 AU**      | All ICS Environments   | Mandatory multi-layered audit trails for Level 3.5 IDMZ security.           |
| **ISA-62443-2-1**       | General Industrial     | Supports forensic visibility across multiple calibration cycles.            |
| **NERC CIP-007-6**      | Power / Utilities      | 3-year evidence retention for cybersecurity incident response and recovery. |
| **EPA 40 CFR Part 141** | Water / Wastewater     | 3-year permit renewal cycles and sanitary survey documentation.             |
| **SIP**                 | All field asset owners | Forensic horizon covering a minimum of three maintenance cycles.            |

## 2. Auditor Agent Fault Tolerance
The **Auditor Agent** is self-monitoring and responsible for the SHA-256 hashing and background integrity scrubbing of
**WORM**. To prevent silent failures, a **Monitor-of-the-Monitor** (MoM) architecture is enforced:

- **Self-Healing**: The Auditor Agent is monitored by a **Hardware Watchdog** ([PVP-02](/docs/commissioning.md)). It must   
  interact with the watchdog independently of the primary Orchestration Agent.
- **Failure Response**: If the Auditor Agent fails or stalls an integrity check, it triggers a hardware-isolated
  `AUDIT_FAILURE` alarm (Blue Square Icon [ □ ]).
- **Process Redundancy**: A secondary **Shadow Auditor** service runs in a separate cGroup to monitor the primary
  auditor's PID and force a restart upon failure.
- **Heartbeat Mismatch**: If the **Shadow Auditor** detects a heartbeat mismatch, it force restarts the primary
  and triggers a hardware-isolated `AUDIT_CRITICAL_FAILURE` alarm.

## 3. Actor Identity & AI Governance
All actions are logged with `actor_id` and `trace_id` to comply with the **90-Day Access Review Schedule**.

- **AI Service Accounts**: Follow the naming convention `SVC_OS_[REGION]_[AGENT_TYPE]_[UUID]`.
- **Credential Rotation**: RSA-4096 signing keys for AI Agents are rotated every 90 days via the secret manager.
- **Review Cycle**: Every 90 days, a human Supervisor must review and sign-off all active AI Service Account 
  permissions.

## 4. Log Schema & Forensic Clarity
Every log entry must remove ambiguity for post-mortem analysis.

| Field                      | Type   | Description                                                                                                                  |
|:---------------------------|:-------|:-----------------------------------------------------------------------------------------------------------------------------|
| `baseline_deviation_limit` | Float  | The Standard Deviation ($\sigma$) limit defined in the Alarm Database.                                                       |
| `active_z_score`           | Float  | The **Real-time Standard Score** that triggered the AI action.  It represents how many $\sigma$ the signal is from the mean. |
| `sha256_hash`              | String | Cryptographic signature of the log block to ensure WORM integrity.                                                           |

## 5. The Unified Audit Schema
All logs are stored in a structured JSON format and replicated to a secure, write-once-read-many (WORM) storage volume.

| Field           | Description                                                                                    |
|:----------------|:-----------------------------------------------------------------------------------------------|
| `timestamp_utc` | High-precision ISO-8601 timestamp (Aligned to PTP Hardware Clock).                             |
| `actor_id`      | Naming convention: SVC_OS_[REGION]_[AGENT_TYPE]_[UUID] (for AI) or Federated JWT (for humans). |
| `trace_id`      | A UUID linking the log entry to a specific LangGraph Reasoning Trace.                          |
| `context_ref`   | JSON object containing `rag_source`, `baseline_deviation_limit`, and `active_z_score`.         |
| `action_type`   | e.g., `INTENT_GENERATED`, `OPERATOR_VETO`, `SETPOINT_CHANGE`, `SECURITY_REJECTION`.            |
| `payload_hash`  | A SHA-256 hash of the command payload to ensure non-repudiation.                               |
| `outcome`       | `SUCCESS`, `FAIL`, or `INTERCEPTED`.                                                           |

## 6. Three-Tier Log Architecture
1. **The Reasoning Log (L3/L4)**: Records the internal **Chain of Thought** and RAG document citations. This is used for 
  **Explainable AI (XAI)** audits.
2. **The Interaction Log (L3)**: Records every HMI action, including which operator viewed which Reasoning Trace.
3. **The Execution Log (L2)**: Records the raw bytes sent over the wire by the Southbound Driver and the physical 
  acknowledgement received from the field asset.

### 6.1 Log Integrity & LGTM Stack Forwarding
- **Cryptographic Signing**: Every log block is digitally signed. A gap in the sequence numbers or a signature 
  mismatch triggers a `SYSTEM_INTEGRITY_FAILURE` alarm.
- **Security Forwarding**: High-criticality security events are forwarded via encrypted syslog to the plant's master SIEM.
- **Log Forwarding**: Logs are streamed in real-time to the LGTM Stack for centralized observability.
- **Retention**: In accordance with industrial compliance, logs are retained for a minimum of 3 years.
- **Verification Checkpointing**: Every command generates a `checkpoint_id` (`GATE_02_SEMANTIC`) and logs `latency_ms` 
  to monitor system determinism.

### 6.2 Verification Checkpoint Logging
Every time a command passes through a gateway, a **Checkpoint Log** is generated:

- **`checkpoint_id`**: (e.g., `GATE_02_SEMANTIC`)
- **`verification_status`**: `PASSED` | `FAILED` | `REDACTED`
- **`latency_ms`**: Time taken to verify. (Used to monitor system determinism).

If a command fails **Gate 2 (Semantic Check)**, a **Tamper Alarm** is sent to the Security Information and Event
Management (SIEM), as this indicates the AI agent attempted an action that violates the hard-coded safety bounds. The
SIEM is the final destination for all **Forensic Traceability** data, and it acts as a "black box" recorder for the plant.

### 6.3 Automated Signature Scheduling
A background process, the **Auditor Agent**, continuously **Scrubs** the **WORM** storage:
1. It re-calculates the SHA-256 `payload_hash` for every entry.
2. It verifies the digital signature of each log block.
3. If any discrepancy is found (indicating data tampering or storage corruption), it immediately triggers a 
   **Forensic Integrity Alarm** in the SIEM.

---

## 7. Disconnected Operations & SIEM Buffer Limits
Because the SIEM acts as the ultimate forensic "black box," the Edge Gateway must handle scenarios where the connection 
to the IT/Enterprise network is severed, preventing real-time log forwarding.

### 7.1 Local Spooling (WORM Buffer)
If the SIEM endpoint is temporarily unreachable, the Edge Gateway uses **TimescaleDB** on its encrypted NVMe drive as a
local spooling buffer. 
- All `actor_id` and `trace_id` records are cached locally with their cryptographic signatures intact.
- The system automatically enters a **Retry-Backoff** state and attempts to reconnect to the SIEM every 30 seconds.

### 7.2 Queue Exhaustion & Safety Lockout Thresholds
To prevent the Edge Gateway from overwriting critical forensic data or executing autonomous actions without an external 
audit trail, strict queuing limits are enforced:
- **80% Disk Capacity (Warning)**: Triggers a local HMI `AUDIT_SPOOL_WARNING`. The system remains in Auto mode.
- **95% Disk Capacity (Gateway Lockout)**: If local WORM storage reaches 95% capacity (or an equivalent 72 hours of 
  disconnected telemetry), the system triggers a **Safety Lockout**.
  - **Action**: The Southbound Driver is immediately suspended. 
  - **State**: The system enters `ORPHAN_MODE` (Safe State) and requires physical, Level 0 manual control to operate 
    field assets. 
  - **Rationale**: An AI agent is strictly forbidden from executing un-auditable actions. If it cannot securely log 
    its reasoning, it cannot act.

### 7.3 Recovery & Replay Handshake
Upon restoration of the SIEM connection:
1. The Edge Gateway initiates a **Replay Handshake**.
2. Buffered logs are forwarded in chronological order, verified by their PTP hardware timestamps.
3. The system remains in `ORPHAN_MODE` until the spool utilization drops below the 80% threshold and the SIEM 
   acknowledges the receipt of the missing sequence numbers.

## 8. Role-Based Log Access (RBAC)
Audit logs contain sensitive intellectual property (RAG vectors) and security data (network traces). Access to the local WORM database and the LGTM stack is strictly governed:
- **Safety Lead / Auditor**: Read-Only access to the full Execution and Interaction Logs.
- **Operator**: Read-Only access limited to the L3 Reasoning Trace for active incidents.
- **AI Agent**: **Write-Only** access to the `intent_outbox`. The AI agent has no read permissions for historical execution logs to prevent contextual poisoning.

## Example Log Entry

```json
{
  "timestamp_utc": "2026-05-04T12:00:00.123456Z",
  "actor_id": "agent_oversight_01",
  "trace_id": "uuid-v4-link-to-langgraph",
  "context_ref": {
    "rag_source": "pump_manual_v2.pdf#page=45",
    "active_z_score": 3.2,
    "baseline_deviation_limit": 3.0,
    "active_loto": false
  },
  "payload_hash": "e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855",
  "verification_checkpoint": "GATE_02_SEMANTIC",
  "outcome": "SUCCESS"
}
```

## Revision History
| Version | Date       | Author    | Description of Change                                                                                                  | Approved By |
|:--------|:-----------|:----------|:-----------------------------------------------------------------------------------------------------------------------|:------------|
| 1.0     | 2026-04-12 | armummert | Initial Baseline for Version 1.0 Release.                                                                              | armummert   |
| 1.1     | 2026-05-04 | armummert | Mapped 3-yr retention to NERC/EPA; Added Shadow Auditor Monitor-of-Monitor; Fixed Z-score nomenclature and LGTM Stack. | armummert   |
