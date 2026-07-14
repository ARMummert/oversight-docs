# Oversight ICS Control Authority Model

## 1. Purpose & Scope
The purpose of this document is to govern control authority between the AI Agent, Human Operators, and deterministic 
logic. The Control Authority Model specifies authority over all field assets (PLCs, VFDs, Actuators, etc.) within the 
Process Asset ecosystem.

## 2. Functional Hierarchy
According to ISA-95 standard, the Control Authority Model is structured as follows:

| Level | Role           | Description                                                                                                                                                                                                            |
|:------|:---------------|:-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| 4     | Enterprise/HMI | High-level monitoring and decision-making interface (Next.js Dashboard).                                                                                                                                               |
| 3.5   | Cognitive      | The **LangGraph Agent** analyzes telemetry to purpose an intent (e.g., modulating pump speed for predictive maintenance).                                                                                              |
| 3     | Supervisory    | Human Operators provide oversight and can acknowledge or override AI-generated intents via the HMI interface. They have the authority to issue commands directly to field assets when necessary.                       |
| 1.5   | Arbitration    | The **Control Authority Model** acts as a gatekeeper, managing the Write-Access Token to so that only one entity has command authority at any given time.                                                              |
| 1     | Deterministic  | Field Assets (PLCs, VFDs) receive validated commands and execute them against their own hardwired interlocks and PID logic.                                                                                            |
| 0     | Physical       | Safety Instrumented Systems (SIS) and Hand-Off-Auto (HOA) switches provide the ultimate authority, capable of physically disconnecting the process from all digital logic in case of emergencies or safety violations. |

## 3. Command Authority Arbitration Logic

### 3.1 Write-Access Token
To maintain integrity and prevent race conditions (two commands from being executed simultaneously), Oversight uses 
a **Write-Access Token**.

> Write-Access Tokens are cryptographically signed by the TPM (Trusted Platform Module) 2.0 to ensure non-repudiation 
> (proving that a statement was made without any interference from someone else) and secure provenance (recording the 
> origin).

- **Ownership**: Only one entity (AI Agent or Human Operator) may have a Write-Access Token at any given time.
- **Requests**: Authority is requested from the Arbitrator. The Arbitrator evaluates the system state before 
  granting the token.
- **Heartbeat Requirement**: The token holder must maintain a continuous cryptographic heartbeat. If the heartbeat 
  fails, the token is immediately revoked and the system enters `ORPHAN_MODE`.

### 3.2 Bumpless Transfer Protocol
To prevent mechanical surges and industrial reliability with transitioning authority (e.g., from AI to Human Operator), 
the system enforces **State Alignment** and **Ramp Constraints**.

- **State Alignment**: The entity acquiring authority must synchronize its intended setpoints with real-time 
  field telemetry before the Token is officially granted.
- **Ramp Constraints**: The Arbitrator enforces safety "ramps" to move parameters to new targets, preventing 
  mechanical surges.

# Control Authority Model: System Operating Modes

The OverSight ICS uses a structured hierarchy of authority modes to ensure deterministic safety, clear command 
provenance, and seamless transitions between AI-driven optimization and human-led safety protocols.

## 4. Authority Mode Matrix

The following table defines the operational boundaries for each system state. These modes are enforced at **Level 1.5 
(The Arbitration Layer)**.

| Mode                 | Token Holder           | Command Path                  | Technical Description                                                                                                                                                                                                  |
|:---------------------|:-----------------------|:------------------------------|:-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `CONTROL_ENABLED`    | **AI Agent**           | AI → Gateway → Field Asset    | **Full Autonomous:** The LangGraph agent holds the active Write-Access Token. Intents are signed by the TPM 2.0 and executed directly.                                                                                 |
| `ADVISORY_MODE`      | **None (Parked)**      | AI → HMI (Human Approval)     | **Human-in-the-Loop:** AI proposes intents; however, the Token is parked at the Arbitrator. Execution requires manual acknowledgment via MFA.                                                                          |
| `MONITOR_ONLY`       | **None**               | Field Asset → AI (Passive)    | **Passive Observation:** The system observes telemetry and logs state. The Arbitrator blocks all outbound "Intent" packets from the AI.                                                                                |
| `AI_LOCKOUT`         | **Human**              | HMI → Field Asset             | **Manual Digital:** AI is logically severed from the control loop. Only the Operator (Level 3) can issue commands via the SCADA/HMI.                                                                                   |
| `MAINTENANCE_MODE`   | **Technician**         | Local Interface (Level 0/1)   | **Local Override:** Remote commands (AI or HMI) are blocked. Authority is restricted to local physical interaction at the asset.                                                                                       |
| `ORPHAN_MODE`        | **PLC (Local)**        | Local Deterministic Logic     | **Comm-Loss Fail-Safe:** Triggered by heartbeat loss. AI/HMI are disabled; the PLC reverts to its local, hardcoded safety logic.                                                                                       |
| `SAFETY_LOCKOUT`     | **SIS**                | Physical Safety Chain         | **Critical Trip:** All digital write-access is destroyed. Requires a physical hardware reset after a safety violation or E-Stop event.                                                                                 |
| `COMMISSIONING_MODE` | **Shared(Supervised)** | AI/HMI → Gateway (Restricted) | **Sandbox Testing**: Commands are permitted but subject to ultra-tight "Semantic Gatekeeper" constraints and manual per-step validation. TPM 2.0 testing key used to sign packets while verifying new hardware spares. |

## 5. Logic Enforcement & Arbitration Rules

### 5.1 Token Priority Precedence
The Arbitrator must always prioritize the mode with the lowest **Purdue Level** to ensure safety over automation:
1. **Priority 1 (Highest):** `SAFETY_LOCKOUT` / `MAINTENANCE_MODE` (Level 0).
2. **Priority 2:** `ORPHAN_MODE` (Level 1).
3. **Priority 3:** `AI_LOCKOUT` (Level 3).
4. **Priority 4:** `MONITOR_ONLY` (Level 3).
5. **Priority 5:** `ADVISORY_MODE` (Level 3.5).
6. **Priority 6 (Lowest):** `CONTROL_ENABLED` (Level 3.5).

### 5.2 Immediate Revocation
Upon transition to `ORPHAN_MODE` or `SAFETY_LOCKOUT`, the Arbitrator is programmed to:
-   Immediately invalidate the current **Write-Access Token**.
-   Instruct the **TPM 2.0** to stop all signing operations for AI-generated packets.
-   Trigger a **Critical Event** log in the TimescaleDB Historian for forensic audit.

## 6. Hardware Root of Trust (TPM 2.0 Integration)
The **TPM 2.0** acts as the hardware root of trust for all command authorization. The following security measures are 
enforced:
-   **Intent Signing:** Every command must be signed by the TPM 2.0 using non-exportable keys.
-   **Platform Attestation:** The TPM verifies the **[SBOM](/docs/sbom_compliance_report.md) (Software Bill of Materials)** and platform state before signing.
-   **Non-Repudiation:** Every command is cryptographically tied to the hardware, creating an immutable audit trail in **TimescaleDB**.

## 7. The Override Hierarchy (Priority Stack)
This stack defines which system component has the ultimate authority during a conflict:

1. **Priority 1 (Absolute): Physical HOA/SIS (Level 0)** - Hardwired switches and safety trips.
2. **Priority 2: Hardware Root of Trust / TPM (Level 0.5)** - Blocks signing if hardware integrity is compromised.
3. **Priority 3: Local PLC Safety Logic (Level 1)** – Deterministic local interlocks on the field asset.
4. **Priority 4: Network Heartbeat / Arbitrator (Level 1.5)** - Forces `ORPHAN_MODE` upon loss of communication.
5. **Priority 5: Human Operator Command (Level 3)** - Manual overrides via HMI/SCADA.
6. **Priority 6: Semantic Gatekeeper (Level 3.5)** - Validates AI intents against allowed physical operating ranges.
7. **Priority 7 (Lowest): AI Agent Proposed Intent (Level 3.5)** - Predictive optimization intents.
