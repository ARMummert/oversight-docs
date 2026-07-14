# Commissioning Guide

This document defines the formal validation procedures for the OverSight ICS Edge Gateway. Successful completion of
these tests is a prerequisite for the **Active Control** or **Passive Monitor** mode of operation.

## Prerequisites
- [Installation Guide](/docs/install.md) must be completed.
- [FAT-01: Hardware & Communication Protocol](fat/FAT-01-hardware_and_communication_protocol.md) must be completed with all tests passing.
- [FAT-02: Functional & Logic Protocol](fat/FAT-02-functional_and_logic_protocol.md) must be completed with all tests passing.
- [FAT-03: Safety and Security Protocol](fat/FAT-03-safety_and_security_protocol.md) must be completed with all tests passing.
- [Site Acceptance Testing (SAT)](/docs/sat/site_acceptance_testing.md) must be completed with all tests passing.

## Phase 0: Pre-Energization & Physical Audit
Before applying the OverSight ICS Edge Gateway, the following industrial pre-commissioning steps must be recorded
in the site log:

- [ ]  **Insulation Resistance (Megger) Testing**: Verify that all new 24 V DC power runs and communication cabling meets 
  minimum mega-ohm requirements.
- [ ] **P&ID Walk-down**: Physically trace the sensor-to-gateway path to ensure the "Digital Twin" tag mapping 
  (e.g., `PUMP_01_FLOW`) matches the physical instrument location.
- [ ] **Loop Checking**: Perform a 4–20 mA loop check to confirm signals generated at the field instrument are correctly
   received at the Field Asset/Gateway without scaling errors.

## Phase 1: PTP & Time Synchronization Verification
Before powering up the OverSight ICS Edge Gateway, the PTP clocks must be verified.

- [ ] **PTP Grandmaster Clock**: Run
```bash
pmc -u -b 0 'GET TIME_PROPERTIES_DATA_SET'
```
to verify the properties of the Sratum 1 PTP Grandmaster clock on the Northbound NIC (NIC 1).

- [ ] **PTP Grandmaster Lock**: Run 
```bash 
  pmc -u -b 0 'GET TIME_PROPERTIES_DS'
  ``` 
  to verify the Sratum 1 PTP Grandmaster
  lock on the Southbound NIC (NIC 2).

- [ ] **Jitter Verification**: Confirm PTP jitter remains $<100\mu s$ on the local Level 2 subnet for a continuous 
  10-minute window.

- [ ] **Boundary Verification**: Check the supervisory scan rate and heartbeat interval. The system must maintain a 
  deterministic timing model with a 100 ms processing window and heartbeat interval, allowing for a 5x safety buffer 
  before the hardware watchdog triggers a safe state.
 
## Phase 2: Hardware, Cgroup Isolation & SBOM Verification
The Oversight ICS Edge Gateway uses cgroups for resource isolation and verifies the software integrity via the 
SBOM manifest. 

- [ ] **Resource Locking**: Verify the Reasoning Agent service is locked to CPU cores 0–1 via `systemd` slices to prevent
      resource exhaustion.
- [ ] **SBOM Integrity**: Verify the running service hashes match the 
      [SBOM Compliance Report](/docs/sbom_compliance_report.md). If hashes do not match, the system must trigger a 
      P1 incident.
- [ ] **Northbound Restriction**: Confirm the firewall blocks all Layer 3 traffic except to the designated Vertex AI
      Endpoint via NIC 1 (Northbound).
- [ ] **Dual-NIC Isolation**: Verify that no IP forwarding is enabled between NIC 1 (Blue/IT) and NIC 2 (Green/OT).

### Timing & Determinism Boundaries
Oversight ICS operates under a deterministic timing model. These boundaries are all for safe supervisory control while 
distinguishing between the field asset's hard real-time tasks and the Edge Gateway's Supervisory tasks. Confirm the
following boundaries are maintained:

- [ ] **Supervisory Scan Rate**: 100 ms processing window 
- [ ] **Heartbeat Interval**: 100 ms mandatory signal to field assets 
- [ ] **Watchdog Timeout**: 500 ms hardware relay trigger 
- [ ] **Safety Buffer (5x)**: This allows for four consecutive missed heartbeats (due to computational jitter or network 
  latency) before the hardware relay opens, forcing a **Safe State**.
- [ ] **PTP Sync**: Verification of Stratum 1 PTP Grandmaster lock on NIC 2 (OT Network).
- [ ] **Dual-NIC Isolation**: Verification that NIC 1 (Northbound) and NIC 2 (Southbound) are on independent subnets 
      with no IP forwarding enabled.

### Edge Gateway Specifications
- [ ] **Hardware**: Industrial PC (IPC) with 4-Core CPU, 16GB RAM.
- [ ] **Power**: Redundant Dual 24 V DC industrial power inputs.
- [ ] **Interfaces**: Dual-NIC (Network Interface Cards) for physical layer isolation.
- [ ] **OS**: Linux (Ubuntu 22.04 LTS or equivalent) with Docker Engine 24.0+.

### Network Segmentation (IDMZ)
| Interface | Zone           | Connectivity                                                                                   |
|:----------|:---------------|:-----------------------------------------------------------------------------------------------|
| **NIC 1** | **Blue (IT)**  | **Northbound**: Connected to the Plant Business Network (Level 4) for UI and LLM API calls.    |
| **NIC 2** | **Green (OT)** | **Southbound**: Connected to isolated OT Network (Level 1, 2). No gateway, no internet access. |

## Phase 3: Power & Physical Power Validation Procedures (PVP)
Before powering up the OverSight ICS Edge Gateway, the physical integrity of the IPC must be verified.

### PVP-01: Redundant Power Failover
1. **Action**: Apply 24 V DC to both V1 and V2 power inputs.
2. **Test**: Disconnect the Primary Rail (V1)
3. **Requirement**: The IPC must remain powered on without rebooting.
4. **Verification**: Confirm **"Primary Power Loss"** is logged in the OverSight ICS HMI log.

### PVP-02: Physical Watchdog Trigger
1. **Action**: Start the `southbound_driver` service to engage the heartbeat.
2. **Test**: Manually terminate the service process (kill -9)
3. **Requirement**: The physical hardware watchdog relay must open within 500ms of heartbeat cessation (matching the
   5x safety buffer).
4. **Verification**: Verify the connected process registers a `Communication_Fault` and enters **Safe State**.

## Phase 4: Installation Verification Pre-Flight
Before beginning any Operational Acceptance Tests (OAT), verify that the following from [Installation Guide](/docs/install.md) are complete:

- [ ] **Physical Mounting**: IPC is DIN rail mounted in the control cabinet with dual 24 V DC rails connected.
- [ ] **Dual-NIC Isolation**:
  - **NIC 1** is verified on the **Blue (IT)** subnet.
  - **NIC 2** is verified on the **Green (OT)** subnet with no default gateway and no internet access.
- [ ] **Compliance Check**: The `compliance_watcher.py` script has returned `INFO: System Compliant` for a 
      continuous 5-minute period.
- [ ] **Clock Sync**: `chrony` (NIC 1) and `ptp4l` (NIC 2) are confirmed stable.

## Phase 5: Semantic Gatekeeper "State of Grace"
The Semantic Gatekeeper is the first service to run on the OverSight ICS Edge Gateway. It performs a critical role 
in validating the integrity of all incoming commands and proposals before they can affect the field assets. Before 
proceeding to functional validation, the Gatekeeper must be verified to be in a "State of Grace" where it is fully 
operational but has not yet blocked any commands.

- [ ] **Unauthorized Write Test**: Attempt to inject a **Forbidden** register write (per [FAT-03 - ST-01](/docs/fat/FAT-03-safety_and_security_protocol.md))
- [ ] **Rejection Logic**: Confirm that the Southbound Driver rejects the write with a `403 FORBIDDEN` error and logs a
      P1 Security Violation event.

## Phase 6: Operational Readiness – Data Layer

- [ ] **Table Loading**: Successfully load the `asset_config` and `forbidden_registers` tables into the production
      environment.
- [ ] **SOE Mapping**: Every asset must be mapped to a **Safe Operating Envelope (SOE)** before software validation.
      Define the mechanical limits (e.g., RPM, Hz) for each asset using [SOP-SAFE-ENVELOPE-01](/docs/sop/SOP-OPS-003_Safe-Operating-Envelope.md).
- [ ] **Persistence**: Attempt to process an intent with an invalid `justification_hash` and verify that it is rejected.
- [ ] **Forbidden List**: Load the **Forbidden Register List** for speed-locks and critical interlocks. 

### `asset_config` Table
The `asset_config` table contains the following information for each asset:

| Asset Type | Unit | Min Setpoint | Max Setpoint | Critical Threshold   |
|:-----------|:-----|:-------------|:-------------|:---------------------|
| Pump       | RPM  | 300          | 3600         | >3800 (over-speed)   |
| VFD        | Hz   | 15.0         | 60.0         | >62.0 (Thermal Trip) |
| Valve      | Pos  | 0%           | 100%         | N/A                  |

[!NOTE] *These limits must be cross-referenced against Master FMEA and the plant's physical relief valve settings.*

## Phase 6.1: RAG Ingestion Verification ([SOP-AI-01](/docs/sop/SOP-AI-002_RAG-Ingestion-and-Vector-Indexing.md))

- [ ] **Document Ingestion**: Upload site-specific manuals to the ingest directory per 
      [SOP-AI-01](/docs/sop/SOP-AI-002_RAG-Ingestion-and-Vector-Indexing.md).
- [ ] **Vector Integrity**: Issue a test query to the Vector DB; verify the system returns a relevant citation with a 
      valid `justification_uuid`.

## Phase 6.2: Operational Readiness - Simulation


## Phase 7: Functional Validation & Loop Checks
Detailed step-by-step validation is located in [VALIDATION_PROTOCOLS.md](/docs/validation_protocols.md).

- [ ] **Bumpless Transfer**: Perform a **Passive to Active** transfer simulation and verify that no disruption to 
      the field asset or heartbeat cessation occurs.
- [ ] **MFA-Training**: Complete operator MFA training and validate secure session creation in the `operator_sessions`
      portal.
- [ ] **Signature Verification**: Attempt to process an intent with an invalid `justification_hash` and verify that it 
      is rejected.
- [ ] **Gatekeeper Block**: Verify high-impact threshold changes (>15%) require two-key bypass and CR form.
- [ ] **Heartbeat Trip**: Manually stop the Southbound Driver; verify the PLC enters **Safe State** within 500ms.
- [ ] **SOP-AI-03 Readiness**: Verify that the operator can successfully execute the steps in 
     [SOP-AI-03](/docs/sop/SOP-OPS-002_Alarm-Triage-and-Response.md) to respond to a $Z$-Score alert.

### 7.1: Software & Middleware Validation Protocols (SVP)
The following tests must be completed before energizing the physical outputs.

- [ ] **Outbox-to-Driver Latency**: Manually insert a **No-Op** intent into the `intent_outbox` table; verify the
Southbound Driver picks up the row and marks it as `EXECUTING` within <50ms.

- [ ] **TPM Signature Verification**: Attempt to process an intent with an invalid `justification_hash`; verify the 
Southbound Driver rejects the command with a `SIGNATURE_MISMATCH` error.

- [ ] **TTL Expiry Logic**: Insert an intent with an expires_at timestamp set in the past; verify the Driver immediately 
moves it to `FAILED` without attempting a PLC write.

- [ ] **Digital Twin Sync**: Verify that the `asset_registry` in the Gateway matches the current UNS (Unified Name Space) 
hierarchy with zero **Orphaned Tags**.

These SVP tests verify that the **Thick Edge** stack is healthy and communicating internally.

| Test ID    | Component        | Procedure                                                       | Acceptance Criteria                                                                                                                              |
|:-----------|:-----------------|:----------------------------------------------------------------|:-------------------------------------------------------------------------------------------------------------------------------------------------|
| **SVP-01** | Container Stack  | Run `docker compose ps`                                         | All services (FastAPI, Timescale, RabbitMQ) show Up (healthy).                                                                                   |
| **SVP-02** | Telemetry Ingest | Check `telemetry_ingest` logs for active OPC-UA/Modbus polling. | Logs show successful OPC-UA subscription with Quality=Good for at least 3 tags. Modbus polling reports <1% timeout rate over a 60-second window. |
| **SVP-03** | RAG Readiness    | Issue a test query to the Vector DB via the local API.          | System returns a relevant citation from the uploaded PDF manuals.                                                                                |
| **SVP-04** | Persistence      | Restart `timescaledb` container                                 | Data integrity is maintained and no loss of historical $Z$-Score baselines.                                                                      |
| **SVP-05** | Signing Logic    | Attempt a **No-Op** intent write                                | Verify the **TPM 2.0** hardware-backs the `justification_hash` signature as defined in the CSMP.                                                 |

### 7.2: Logic & Discrete I/O Validation 
This section focuses on data integrity and the deterministic handshake with the **Logic Solver**.

- [ ] **Register Point-to-Point**: Force a valve in the PLC (e.g., 40001 = 1234) and verify the Gateway reads the
identical integer without bit-shift or scaling errors.
- [ ] **Heartbeat Trip**: Manually stop the Southbound Driver service and verify the PLC `OVERSIGHT_ACTIVE` bit drops
and the asset enters **Safe State** within 500ms.
- [ ] **Discrete Mapping**: Toggle the physical switch (Hand-Off-Auto) and verify the status change is reflected in the
Gateway's `asset_status` topic within 100ms.
- [ ] **Write Authorization**: Attempt to write to a register on the **Forbidden Register List** and verify the
Southbound Driver blocks the write and logs `SECURITY_VIOLATION`.

### 7.3: VFD & Rotating Equipment
This section focuses on the mechanical safety, frequency limits, and protective behavior of motor drives.

- [ ] **Rotational Check**: Issue a 5Hz **Forward** command via the Gateway and physicall verify the motor shaft rotates
in the direction defined by the P&ID.
- [ ] **Clamping Logic**: Attempt to send a speed setpoint below the `MIN_FREQ` (e.g., 5Hz on a 15Hz pump) and verify
the VFD firmware clamps the output to the baseline minimum.
- [ ] **Communication Loss (VFD Specific) Disconnect the Ethernet cable from the VFD and verify the driver performs an 
immediate **COAST-TO-STOP** rather than ramp down.
- [ ] **Ramp Rate Audit**: Command a speed change from 20Hz to 50Hz and verify the **Time to Steady State** matches
the `ramp_rate_sec` defined in the [Technical Contract](/docs/technical_contracts.md).
- [ ] **Torque Limit Verification**: Verify that the Agent cannot exceed the hardware torque limits set in the VFD
parameters during high-load scenarios.

## Phase 8: Operational Acceptance Tests (OAT)
These tests verify the **Semantic Gatekeeper** and **Consciousness Loop** logic function correctly in a live-data 
environment.

| Test ID    | Description            | Test Procedure                                                           |                                                     Acceptance Criteria                                                     |
|:-----------|:-----------------------|:-------------------------------------------------------------------------|:---------------------------------------------------------------------------------------------------------------------------:|
| **OAT-01** | Gatekeeper Block       | Attempt to write a setpoint to a tag on the **Forbidden Register List**. |                               Semantic Gatekeeper returns `403 FORBIDDEN`; write is blocked.                                |
| **OAT-02** | Low-Impact $\Delta$    | Issue a threshold change proposal of <5%                                 |                        System Auto-Approves; entry created in `threshold_history` as `SYSTEM_AUTO`.                         |
| **OAT-03** | Medium-Impact $\Delta$ | $\Delta$Issue a threshold change proposal of 10%.                        |                             System stages proposal; requires Single-Operator sign-off via HMI.                              |
| **OAT-04** | High-Impact Block      | Issue a threshold change proposal of >15%.                               |                               Gatekeeper Blocks command; requires Two-Key bypass and CR Form.                               |
| **OAT-05** | RAG Traceability       | Inspect a committed threshold change in the Audit Log.                   |                        Entry contains a valid `justification_uuid` linked to a documentation trace.                         |
| **OAT-06** | Shelf Capacity         | Manually shelf 5 active alarms and attempt to shelf a 6th.               |                               System blocks 6th and issues `SHELVED_ALARM_CAPACITY_EXCEEDED`.                               |
| **OAT-07** | SOP-SHELVED-01         | Trigger OAT-06 and require operator to clear a slot via SOP              | Operator successfully restores capacity using the triage steps in [SOP-SHELVED-01](/docs/sop/SOP-OPS-004_Shelved-Alarms.md) |

## Phase 9: Site Acceptance Test (SAT)

| Test ID    | Description         | Test Procedure                                                                                                                               | Acceptance Criteria                                                                       |
|:-----------|:--------------------|:---------------------------------------------------------------------------------------------------------------------------------------------|:------------------------------------------------------------------------------------------|
| **SAT-01** | End-to-End Anomaly  | Trigger a known mechanical transient (e.g, bypass valve cycle).                                                                              | HMI displays a "Real-Time Drift" alert within <2 seconds.                                 |
| **SAT-02** | Remote Notification | Trigger a High-Severity alert.                                                                                                               | Verification that the alert was successfully pushed to the Level 4 Business Network.      |
| **SAT-03** | SOP-AI-02 Readiness | Operator performs a **Threshold Review** using the steps in [SOP-AI-02](/docs/sop/SOP-AI-003_Threshold_Adaptation_Review_and_Commitment.md). | Operator successfully validates a proposal using the provided RAG reasoning (Trace UUID). |
| **SAT-04** | Bumpless Transfer   | Switch from **Passive Monitor** to **Active Control** mode during operation.                                                                 | No disruption to PLC logic; Southbound Driver maintains the heartbeat.                    |
| **SAT-05** | Security Integrity  | Run `compliance_watcher.py` while NIC 1 & 2 are both connected.                                                                              | Script returns `INFO: System Compliant` with no unauthorized bridging detected.           |

## Phase 10: Baseline Training (48-72 hours)
The system must remain in **Passive Monitor Mode** for a minimum of 48 hours to establish the statistical baseline for
the $Z$-Score anomaly detection.

## Phase 11: Final Commissioning Sign-Off
I hereby certify that the OverSight ICS Edge Gateway has passed all safety and functional validations.

### Pre-Flight Confirmation:

[ ] All Dual-NIC IP addresses are static and verified.

[ ] PTP Grandmaster clock is stable and synced with Level 2 assets.

[ ] The **Forbidden List** has been loaded and verified by the Safety Lead.

[ ] **Loop Checks** and **P&ID Walk-down** completed and logged.

### Required Authorization Signatures:
**Lead OT Engineer**: __________________________ Date: _______________  
*Certifies all loop checks, PTP syncs, and environment provisioning are stable.*

**Safety Lead (Witness)**: ___________________________ Date: _______________  
*Certifies that the Forbidden Register List matches the site SOE and FMEA.*

**Plant Manager**: ___________________________ Date: _______________  
*Final authorization to transition the system from Passive Monitor to Active Control.*

**IT Network Administrator**: ___________________________ Date: _______________  
*Certifies the IDMZ Northboundn NIC configuration and verifies enterprise network isolation/firewall rules.*

## Revision History
| Version | Date       | Author    | Description of Change                                                 | Approved By |
|:--------|:-----------|:----------|:----------------------------------------------------------------------|:------------|
| 1.0     | 2026-04-10 | armummert | Initial creation of Commissioning Document                            | armummert   |
| 1.1     | 2026-04-24 | armummert | Document reorganization and timing and determinism boundaries updated | armummert   |
