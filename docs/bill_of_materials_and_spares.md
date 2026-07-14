# Oversight ICS Sofware Bill of Materials and Spares

## 1. Purpose and Scope
This document tracks the verified software components, firmware versions, and approved physical hardware spares for 
Oversight.  Any component not listed here is considered **Untrusted** and will be blocked by the **Level 1.5 
Arbitrator** and **TPM 2.0** policies.

## 2. Software Bill of Materials (SBOM)
The following software stack is measured during the TPM 2.0 boot sequence (PCR Measurements).

| Component                  | Version      | Hash/Source     | Purpose                                        |
|:---------------------------|:-------------|:----------------|:-----------------------------------------------|
| **LangGraph Orchestrator** | v0.1.0-alpha | Custom (TBD)    | Cognitive Agent logic & state management.      |
| **Semantic Gatekeeper**    | v0.1.0-alpha | Custom (TBD)    | Deterministic boundary checking of AI intents. |
| **Python Runtime**         | v3.11-slim   | Official Docker | Base environment for Agent execution.          |
| **TPM2-TSS**               | v4.0.1       | OpenSource      | TCG software stack for TPM interaction.        |
| **TimescaleDB**            | v2.13.0      | TimescaleDB     | Telemetry historian and immutable audit log.   |
| **Mosquitto MQTT**         | v2.0.18      | Eclipse         | Message broker for Level 1.5 - Level 3 comms.  |

### 2.1 Software Integrity Validation
* **Measurement:** During development, `TBD` hashes will be replaced with SHA-256 sums of the compiled binaries/scripts.
* **PCR Policy:** PCR 10 will be used to store the measurement of the `Semantic Gatekeeper` to ensure it hasn't been 
  tampered with before a Write-Access Token is issued.

## 3. Approved Field Asset Spares (Hardware)
Only the following Hardware IDs/Models are authorized to be integrated into the control loop.

| Part Name        | Manufacturer             | Model Number  | Spec / Firmware           |
|:-----------------|:-------------------------|:--------------|:--------------------------|
| **Edge Gateway** | Raspberry Pi / Advantech | 5 / UNO-2271G | 8GB RAM / TPM 2.0 via SPI |
| **Primary PLC**  | Opto 22 (Example)        | GRV-EPIC-PR1  | Firmware v3.5.x           |
| **VFD**          | Danfoss                  | VLT-FC302     | Modbus TCP Interface      |
| **Power Supply** | Phoenix Contact          | QUINT4-PS     | 24VDC / 10A               |

## 4. Spares & Replacement Policy
- **Firmware Lock:** Any replacement hardware must have its firmware flashed to the version listed in Section 3 
  before commissioning.
- **Registration:** New hardware must have its unique UUID/MAC address registered in the `Asset_Registry` table in 
  TimescaleDB before the Arbitrator will acknowledge it.

## 5. Maintenance & Inventory
- **Critical Spares:** One (1) Primary PLC, one (1) Edge Gateway, and one (1) TPM SPI Module must be kept in anti-static 
  storage on-site.
- **Audit Cycle:** Software versions are to be reviewed quarterly. "TBD" versions must be finalized before the system 
  moves from `COMMISSIONING_MODE` to `CONTROL_ENABLED`.

## 6. Supply Chain Custody & Anti-Tamper Verification
To prevent hardware implants or firmware tampering during transit, all incoming replacement hardware must undergo 
strict supply chain custody validation before being introduced to the Level 3.5 IDMZ or the OT network:

- **Visual & Physical Inspection**: 
  - Verify that all vendor tamper-evident seals on the shipping container and anti-static bags are intact.
  - Any hardware arriving with broken seals must be quarantined and marked `UNTRUSTED`.
- **Hardware Root of Trust (TPM Verification)**:
  - For Edge Gateways, extract the TPM Endorsement Key (EK) certificate via an isolated, air-gapped terminal.
  - Cryptographically verify the EK certificate against the hardware manufacturer's published root CA.
- **Firmware Cryptographic Hash Validation**:
  - Before provisioning a new PLC or VFD, dump the factory firmware and calculate its SHA-256 hash.
  - Compare this hash strictly against the vendor's officially published manifest. If the hash does not match, the asset 
    must not be connected to the Level 2 control loop.
- **Cold Room Provisioning**: 
  - All hashing and verification must occur in an isolated staging environment (Level 1 cold loop) prior to live network 
    registration.