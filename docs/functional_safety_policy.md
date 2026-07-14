# Oversight ICS Functional Safety Policy (FSP) 

## 1. Purpose & Scope
The Functional Safety Policy (FSP) defines the technical and procedural requirements for maintaining physical safety 
within the OverSight ICS ecosystem. It specifically addresses the **Independence of Protection Layers (IPL)**, 
ensuring that AI-driven reliability logic can never override or interfere with the **Safety Instrumented System (SIS)**.

## 2. Safety Dominance & Independence
In accordance with IEC 61511, OverSight is classified as a **Non-Safety-Related Reliability Layer**.

- **Physical Independence**: The AI Reasoning Agent (Python/Docker) runs on independent hardware (Edge Gateway) from 
  the SIS Logic Solver (Safety PLC).
- **Hardwired Priority**: All safety interlocks (E-Stops, high-pressure trips, etc.) are hardwired or programmed in a 
  Safety field asset. AI \"Intents\" are physically incapable of forcing a safety-interlocked register to a dangerous 
  state.
- **Write-Restriction**: The AI is strictly prohibited from writing to any register tagged as \"Safety-Critical\" in 
  the Digital Twin.

### 2.1 Out-Of-Bounds Safety Verification
Oversight implements an **Out-Of-Bounds Safety Verification** to prevent **State Spoofing** or **Man-in-the-Middle** 
attacks where a compromised UNS or MQTT Broker provides safe feedback to the HMI while the physical hardware is in
a fault state.

### Risk: 
If the HMI only receives safety status via the standard `Telemetry` topic, a logic compromise at the
middleware or broker level could mask a hardware trip, leading the operator to believe the system is in a **Nominal** 
state when it is in fact in a **Faulted** state.

### OOB Requirement – Dedicated Read-Only Heartbeat:
The HMI and the Southbound Driver must maintain a **Dedicated Safety Path** that operates independently of the 
Reasoning Agent's data processing loop:

1. **Dedicated Tag Mapping**: A specific `Safety_Healthy` bit must be mapped in the field asset. This bit toggles
   at the frequency of 1Hz directly from the field asset's hardware clock.
2. **Direct Read Protocol**: The HMI will pull this status via a read-only, high-priority subscription that bypasses
   the RAG and Reasoning nodes in LangGraph.
3. **Visual Deadman Switch**: If the HMI does not detect a state change on the **OOB Bit** for >3 seconds, it must
   immediately trigger a **Communication Integrity Alarm** and overlay a grey **Data Stale** watermark across all 
   AI-suggested intents.

#### Implementation Standards:

1. This bit must be generated in the Safety field asset (Level 1) and be unreachable by any **Write** command from 
   the Gateway.
2. The OOB path must use a different MQTT Topic Branch (e.g., Enterprise/Site/Safety/Heartbeat) with stricter 
   **Access Control Lists** (ACLs) to prevent unauthorized access instead of using the standard `Telemetry` topic.

## 3. The Heartbeat & Watchdog Protocol
To prevent a \"Frozen Brain\" scenario (where the AI hangs but the system remains in its last commanded state), 
OverSight implements a bidirectional watchdog.

### 3.1 Gateway-to-PLC Heartbeat
- **Mechanism**: The Edge Gateway toggles a dedicated \"Heartbeat Bit\" in the Level 2 Controller every 500ms.
- **Failure Condition**: If the Level 2 Controller does not detect a state change within **1500ms (3 cycles)**, it 
   assumes the Edge Gateway has failed.
- **Action**: The PLC immediately severs the AI’s write-authority and reverts to **Local Autonomous Mode**.

### 3.2 Agent Deadman Switch (Command TTL & Clock Integrity)
While the Southbound Driver Heartbeat monitors the system availability, this switch enables individual (intents) to be 
validated only within a temporal window.
- **Command TTL**: To prevent the execution of stale logic or **ghost** commands, all intents have a 
  **Time-To-Live (TTL)** of 2000 ms.
- **Contract**: `Expiration_Time = created_at + 2000ms`
- **Validation**: The Southbound Driver will reject any command where the timestamp is $>2000$ms older than the current 
  system clock, preventing the execution of \"stale\" logic.

### 3.2.1 Clock-Skew Mitigation & PTP Synchronization
To prevent safety logic errors caused by implementation-level drift, the Southbound Driver must verify the **Truth** of 
the command's timestamp before execution.
- **Reference Clock**: The Southbound Driver retrieves the local hardware system clock, which is strictly synchronized 
  with the PTP Grandmaster clock.
- **Drift Threshold**: The Southbound Driver compares its local PTP clock against the intent's created_at timestamp.
- **Rejection Rule**: If a drift >100ms is detected between the Reasoning Agent's timestamp and the Southbound Driver's 
  local clock, or if the current time exceeds the TTL, the command is immediately rejected.
- **Error Response**: The system must return a `401 SECURITY_CLOCK_SKEW_ERROR` and log the event into the **WORM** 
  audit trail.

### 3.3 Agent-to-Driver "Internal" Watchdog (Stall Check)
Beyond the physical field asset heartbeat, a software watchdog exists between the **Reasoning Agent** and 
the **Southbound Driver**.
- **Function**: The Reasoning Agent must write a "Reasoning Alive" token to a shared memory segment or Redis key every 
  2 seconds.
- **Stall Detection**: If the Southbound Driver detects that the token is >5 seconds old (even if the Python process is 
  technically "running"), it will:
    1. Set the HMI status to `AGENT_STALL_DETECTED`.
    2. Ignore all new records in the `intent_outbox`.
    3. Transition the system to **Advisory Only** mode.

## 4. Safe-Hold States
If the AI detects an anomaly it cannot resolve with high confidence, or if a system failure occurs, the asset must 
transition to a **Safe-Hold State**.

| Asset Type         | Safe-Hold Action       | Engineering Rationale                                             |
|:-------------------|:-----------------------|:------------------------------------------------------------------|
| **Pumps/Motors**   | Maintain Current Speed | Prevents hydraulic hammer or sudden pressure loss in the header.  |
| **Valves**         | Fail-Last-Position     | Maintains current process stability to avoid surge or cavitation. |
| **Heaters**        | Force 0% Output        | Standard fail-safe to prevent thermal runaway during logic loss.  |
| **Standby Assets** | Available/Ready        | Ensures standby equipment is unlocked for manual operator start.  |

## 5. Lockout/Tagout (LOTO) & Maintenance Bypass
Oversight respects a physical **Hardware-To-Software** handshake, which guarantees the safety of field personnel.

### 5.1 Local/Hand Selector Logic (LOTO)
Every asset supervised by the Southbound Driver (NIC 2) must monitor a physical "Maintenance Bit" (typically mapped 
to a Local/Remote selector switch on the MCC).

- **Auto/Remote**: The AI Agent is permitted to issue intents through the Southbound Driver.
- **Hand/Local**: The Southbound Driver detects the status bit and enters **Hard Bypass Mode**.
  - All incoming AI intents for that specific `asset_id` are rejected instantly with a 
    `403 Forbidden: Maintenance Bypass` error.
  - The AI reasoning agent is notified that the asset is "Unavailable for Control."
  - This logic is hard-coded at the Driver level and cannot be overridden by the AI or the HMI.
  - **Bumpless Transfer on Recovery**: Upon transition from Hand to Auto, the AI Agent must perform a 'State Sync' pass
    for 30 seconds to observe current process values before attempting any control intervention.

### 5.2 Recovery & Re-Engagement (Bumpless Transfer)
Safe transition governance for moving from **Manual/Hand** to **Auto** control.
- **30-Second State Sync**: Upon re-entering **Auto** mode, the system performs a mandatory **30-Second State Sync**.
  During this time, the Reasoning Agent observes the current process variables to align its internal model with the 
  physical state.
- **HMI Visual Requirement**: During this synchronization period, the HMI dashboard must display the `RESYNC_ACTIVE` 
  state.
- **Bumpless Execution**: The first command issued after a resync must use the current field value as the baseline to
  prevent sudden process surges (e.g., matching current VFD frequency rather than jumping to a stale setpoint).

### 5.2 Error Propagation
When the Southbound Driver detects a High bit on the physical Local/Hand register:
1. It immediately flushes the intent buffer for that asset.
2. It returns a `FORBIDDEN_HARDWARE_LOTO` status to the API Gateway.
3. The HMI must display a "Physical Lockout Active" warning, preventing further confirmation attempts until the physical 
   switch is returned to "Auto."

### 5.3 Software Tagout 
In addition to physical LOTO, operators can issue a **Software Tagout** command through the HMI which sets an 
`is_tagged_out` flag in the Postgres metadata, preventing the AI from including that asset in its failover calculations.

## 6. Pessimistic Confirmation (The Handshake)
OverSight utilizes a \"Pessimistic\" execution model for all failover events to ensure the physical world matches the 
digital intent:
1.  **Intent**: AI proposes a standby pump start.
2.  **Validation**: Southbound Driver checks hardware interlocks and TTL.
3.  **Execution**: Command is sent to the PLC via the secure conduit.
4.  **Verification**: AI waits for a physical \"Running\" feedback signal from field instrumentation.
5.  **Confirmation**: Only after the physical signal is received is the failover marked as \"Success.\" If no feedback 
    is received within **5 seconds**, the AI triggers a **Critical Safety Alarm** and hands control to the human 
    operator.

## 7. Manual Override (Veto Power)
The human operator retains absolute authority over the system at all times.
- **The \"Kill Switch\"**: A global \"AI Lockout\" button is present on every HMI screen. Activating this immediately 
  clears the Southbound Driver's intent buffer.
- **Physical Tag-Out**: Any asset tagged for maintenance (LOTO) in the Digital Twin is automatically ignored by the 
  LangGraph agent, regardless of its health metrics.

## 8. Pre-Execution Compliance Scan
Before the Southbound Driver writes to the PLC:
1. **Integrity Check**: Re-verifies the `payload_hash` against the original `trace_id`.
2. **Safety Check**: Checks if the target register is on the **Forbidden** list (SIS-Controlled).
3. **LOTO Check**: Confirms the physical maintenance bit is `LOW`.

## 9. Automated Safety Validation (Scheduled Watchdog Trip)
To guarantee the safety interface is functioning properly, the system performs a **Scheduled Watchdog Trip** during 
low-risk intervals (or as part of the Commissioning Phase).

**CRITICAL SOP REQUIREMENT**: 
Any procedure involving a manual or scheduled trip of the safety heartbeat must adhere to the following requirements:

1. **Pre-Test Condition**: The operator must confirm the plant is in a **controlled**, non-critical state 
   (Idle or Maintenance) before initiation.
2. **Operator Notification**: All HMI terminals must display a 60-second countdown: `WARNING: SCHEDULED SAFETY TRIP INITIATING`.
3. **Procedure**: The Gateway intentionally stops the Heartbeat Bit for 2 seconds.
4. **Positive Recovery**: A manual **Safety Reset** on the HMI is required to re-engage the heartbeat and exit the tripped state.
   error within the defined timeout. The system must remain in a **Safe State** until a manual **Safety Reset** is 
   performed by an authorized operator.
5. **Log Requirement**: This test must be logged with a `type: SYSTEM_VALIDATION` tag to prove periodic safety testing 
   for ISO 9001/IEC 61511 compliance.
6. **Procedure**: The Gateway intentionally stops the Heartbeat Bit for 2 seconds.
7. **Success Criteria**: The field asset must transition to **Safe State** and the Southbound Driver must return a 
   `COMM_FAULT`

For a step-by-step execution guide, see the 
[Scheduled Watchdog Validation & Recovery Procedure](/docs/sop/SOP-09-Scheduled-Watchdog-Validation-and-Recovery-Procedure.md)

## 10. Hardware Safe State Parameters

### 10.1 Asset-Specific Safe State Conditions
The mechanism for detecting an Oversight system failure differs based on the communication capabilities and safety
firmware of the target asset.

### 10.1.1 PLC Watchdog (Logic Solver)
For Programmable Logic Controllers (PLCs), the safety interface relies on a bidirectional **Heartbeat** Bit.
- **Mechanism**: The Southbound Driver toggles a dedicated boolean register ( `Oversight_Heartbeat_Bit` ) every 100ms.
- **PLC Logic Requirement**: The PLC is programmed with a **500ms Watchdog Timer**.
- **Trip Condition**: If the bit stops toggling for >500ms, the PLC must autonomously drop the `OVERSIGHT_ACTIVE` status
  and transition the process to the **Safe State** defined in section 10.2.

### 10.1.2 VFD Watchdog (Rotating Equipment)
For Variable Frequency Drives (VFDs), the safety interface relies on a **Communication Loss Timeout** which is enforced
at the firmware level.
- **Mechanism**: The Southbound Driver maintains a constant cyclic exchange (e.g., Modbus/TCP or Ethernet/IP) with the
  VFD.
- **VFD Configuration**: The VFD must be configured with a **Communication Loss Action** (e.g., Parameter 6.03 in many
  standard drives) set to **COAST-TO-STOP**.
- **Trip Condition**: If the VFD does not receive a valid **Keep-Alive** or **Write** command from the Gateway within 
  500ms, the drives internal firmware must bypass all external speed setpoints and intiate the **COAST-TO-STOP** safe
  state. 

### 10.2 Safe State Parameters
In the event of an OverSight system failure, the field asset logic enforces the following physical conditions:
- **Pumps**: Transition to OFF.
- **Valves (Normally-Closed)**: Transition to CLOSED.
- **Valves (Normally-Open)**: Transition to OPEN.
- **Valves (Modulating)**: Transition to FAIL-LAST-POSITION (as defined by P&ID)
- **VFDs**: **COAST-TO-STOP**. Immediate de-energization to prevent mechanical stress from active ramping during a 
  watchdog trip.
- **HMI**: Displays `SYSTEM FAULT: OVERSIGHT OFFLINE - MANUAL OVERRIDE ACTIVE`

## Revision History
| Version | Date       | Author    | Description of Change                                                                        | Approved By |
|:--------|:-----------|:----------|:---------------------------------------------------------------------------------------------|:------------|
| 1.0     | 2026-04-12 | armummert | Initial Baseline for Version 1.0 Release.                                                    | armummert   |
| 1.1     | 2026-04-28 | armummert | Audit Fixes: Watchdog self-test SOP, Clock-skew mitigation, HMI resync state, and Safe State | armummert   |
